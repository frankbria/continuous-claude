# Automated PR Review + Auto-fix Architecture
This document describes an architecture for implementing the workflow you specified (PR created → automatic app reviews produce comments → agent waits, reviews, decides, fixes, resolves conversations, runs final checks, merges, deletes branch, continues). This is a design document only; no code changes are included here.

## Goals (restating the workflow)
1. PR is created.
2. Connected apps (CodeRabbit, Claude, etc.) automatically review and post comments on the PR.
3. The automation waits up to a configurable window (e.g., 20 minutes) for those comments to appear.
4. The agent collects and analyzes all comments, independently decides whether to fix each suggestion or intentionally ignore it.
5. The agent applies fixes (where chosen) to the PR branch and pushes updates.
6. For every review thread the agent either:
   - implements the fix and marks the conversation resolved, or
   - explains why it intentionally ignored it and marks the conversation resolved.
7. The agent triggers CI/checks or waits for final automated checks from Claude and other integrators.
8. If all checks pass and no outstanding required review requests exist, the agent merges the PR (with chosen merge strategy).
9. Delete the branch.
10. Continue to next instruction/iteration.

## High-level components
- Orchestrator process: the agent runner (existing continuous_claude script or an external orchestrator) that owns the overall loop.
- Git working copy(s): one worktree per parallel agent instance (existing worktree support).
- GitHub APIs / gh CLI: to read PR data, comments, create reviews, resolve threads, close or merge PRs, and push branches.
- Claude (and other LLM-based) agents: produce code edits, commit messages, decisions.
- CI system (GitHub Actions or other) and third-party bots (CodeRabbit) that produce review comments/checks.
- Shared notes/state: SHARED_TASK_NOTES.md plus optional metadata (state file) to keep continuity between iterations.
- Audit/logs: persistent logs for visibility and debugging.

## Data model and state
- SHARED_TASK_NOTES.md — human/AI visible notes persisted on repo for next run (already present).
- per-PR state (volatile or persisted):
  - pr_number, pr_branch, last_synced_at, comment_snapshot_id, decision_snapshot_id
  - review_threads[] with: id, file, line, author, body, category, severity, decision (fix/ignore/ask-human), status (open/resolved), patch_ref (commit/patch)
- local metadata file (optional): `.continuous-claude/pr-state-<pr>.json` stored per-worktree to persist progress across restarts.

## Sequence diagram (logical steps)
1. PR created triggers agent run (manual, cron, or webhook).
2. Orchestrator notes PR number & branch; clones/pulls repo into a worktree and checks out the PR branch.
3. Orchestrator waits for review comments:
   - Poll GitHub for review comments/review threads (GraphQL timeline or REST, or `gh pr view --json reviewThreads,comments,reviews`) for up to configured window (20 minutes default).
   - Use exponential backoff but poll fairly frequently (e.g., 10–30s) to catch near-real-time comments.
   - Stop early if a sentinel condition is met (e.g., all connected apps have finished, or configured "no more reviewers" indicator).
4. After wait window or early completion, collect all review threads and new comments produced by bots during the window.
5. Normalize comments: group by file + range/line + thread id, extract suggested changes (code blocks, diffs, or textual suggestions).
6. Classify each comment:
   - Parse suggestions vs informational comments.
   - Assign severity (critical / recommended / minor) using rule-based heuristics or a lightweight classifier (e.g., keyword matching + LLM classification).
7. For each comment, agent runs independent analysis:
   - Create a decision (fix / ignore / escalate-to-human).
   - Record rationale text (short).
   - If `fix`, generate a concrete patch (one of: line edit, hunk patch, full-file rewrite) using Claude with a prompt that includes:
     - The repo context (relevant file(s) content or minimal context)
     - The comment text and thread history
     - The coding guidelines and tests
     - SHARED_TASK_NOTES.md context
8. Apply patches:
   - Create a single commit per PR iteration or per logical fix (configurable).
   - Commit on the same PR branch (not a new branch) and push to origin to update the PR.
9. Resolve review threads:
   - For fixed items: mark thread resolved (GraphQL `resolveReviewThread` mutation or gh API) and post a short comment like "Applied fix — see commit <sha>."
   - For intentionally ignored items: post a comment explaining rationale and then resolve the thread.
   - For items requiring human input: add a comment requesting clarification and leave thread open or label PR accordingly.
10. Trigger or wait for CI:
    - If any checks are configured, wait for them the same way the existing script does (reuse `wait_for_pr_checks` semantics).
11. Final auto-review with Claude (optional):
    - Run a final Claude check prompt that inspects the PR diff and validates that required checks are addressed.
12. Merge PR:
    - If checks success and approvals as appropriate, merge using configured strategy (squash/merge/rebase).
13. Delete branch and perform cleanup (local branch deletion, worktree cleanup).
14. Continue orchestration (next tasks/iterations).

## Concrete API/CLI operations to use
- Listing and polling comments/threads:
  - GraphQL: Pull `pullRequest.reviewThreads` and `timelineItems` for deeper context and ability to resolve threads.
  - REST / gh: `gh pr view <pr> --repo owner/repo --json reviewDecision,reviewRequests,comments,commits,headRefName` and `gh api` endpoints for comments and reviews.
- Resolve threads:
  - GitHub GraphQL mutation: `resolveReviewThread` on `PullRequestReviewThread`.
  - `gh api graphql -f query='...' -f variables='...'`
- Apply fixes:
  - Locally: edit file(s), run `git add` `git commit` `git push`.
  - Optionally create patch files with `git format-patch` or `git apply`.
- Posting comments and reviews:
  - `gh pr review --repo ... --comment --body "..."` or `gh api` to post review comments.
- Waiting for checks:
  - Reuse `gh pr checks` or `gh pr view --json` polling code already present.
- Security: use GitHub App or least-privilege PAT scoped to repo: `contents`, `pull_requests`, `checks`, `metadata`.

## Decision logic (fix vs ignore)
- Prioritization factors:
  - Severity of comment (critical vs minor).
  - Confidence of automated fix (LLM-provided test results, static checks).
  - Risk of change: small hunk vs large refactor.
  - Whether change touches critical areas (security, tests, CI).
- Policy (example):
  - If comment severity == critical => fix automatically if patch applies cleanly and tests pass locally or CI is green.
  - If severity == recommended/minor and change is low-risk => fix automatically.
  - If change is large or risky => create a draft commit or open an issue/ask human (escalate).
- Rationale recording:
  - For each fix/ignore decision, store a short rationale in SHARED_TASK_NOTES.md and attach a comment to the thread.

## Prompt engineering and context windows
- When asking Claude to produce a patch include:
  - The specific file snippet (±N lines), or entire file if small.
  - The exact code suggestion extracted from the comment.
  - A short rubric (style, tests, performance constraints).
  - A request to produce a unified diff or a patched file that can be applied automatically.
- If patches are large, request step-by-step fixes: "Make minimal change to satisfy comment."

## Concurrency, idempotency, and race conditions
- Use per-PR state to record which threads processed and which commit produced the resolution to avoid double-processing.
- Use optimistic concurrency:
  - Before applying a patch, check `git rev-parse origin/<branch>` or `gh pr view` to ensure branch head hasn't changed since snapshot. If changed, re-sync and re-run classification for new comments.
- For multiple agents running concurrently, lock PR processing (via ephemeral file in a central store or GitHub Check / label indicating "processing by X").

## Timeouts and retries
- Wait-for-comments window: configurable (default 20 minutes).
- CI wait window: reuse existing 30-minute wait with backoff.
- Retry policy for transient errors (API or push errors): retry 3 times with backoff.
- Abort policy: if 3 consecutive errors occur (existing behavior), stop and report.

## Error handling and fallbacks
- If patch application fails due to merge conflict:
  - Option A: try a 3-way merge with contextual LLM assistance.
  - Option B: post comment on PR asking for human resolution and mark corresponding threads accordingly.
- If required checks fail after fix:
  - Post comment summarizing failures, optionally revert fix commit or open a follow-up PR.
- If GraphQL mutation to resolve thread fails:
  - Leave a comment explaining why the thread remains unresolved and include the commit/reference.

## Security & permissions
- Use a GitHub App or token with minimal scopes (repo:contents, pull_requests, issues for comments, checks).
- Avoid granting CLAUDE or any LLM direct push permissions; the orchestrator performs git operations locally.
- Keep secrets (PATs) in environment variables or CI secret stores, not in repo.
- Sanitize any third-party comment contents before passing to an LLM (remove embedded secrets, URLs to private content).

## UX and observability
- Emit logs for each processed thread: id, author, decision, commit sha if fixed.
- Update SHARED_TASK_NOTES.md with a summary of decisions for human-readable audit trail.
- Optionally add commit message conventions: `[auto-fix] <short> (#<pr>)` to make it easy to find auto commits.
- Add a final PR comment summarizing what was applied/ignored and why.

## Implementation roadmap (phases)
1. Design & tests (this doc + small prototype for collecting comments).
2. Collection & waiting:
   - Implement a poller that captures comments/reviewThreads and snapshots them.
3. Classification & decision engine:
   - Build a small rule-based classifier; augment with LLM classification if desired.
4. Patch generation:
   - Integrate Claude prompt templates to output unified diff or file edits.
5. Apply & push + resolve threads:
   - Implement safe apply path (check branch head; create commit; push).
   - Use GraphQL to resolve threads and post rationale comments.
6. CI integration:
   - Reuse existing wait_for_pr_checks logic for consistency.
7. Merge and cleanup:
   - Merge with existing merge logic; delete branch and local artifacts.
8. Hardening:
   - Add locking, idempotency, retries, observability.
9. Optional: human-in-the-loop mode (notify and await human approval for high-risk decisions).

## Example prompt templates (conceptual)
- "Collect review threads and provide a short JSON array with: thread_id, file, start_line, end_line, suggestion_text, severity. Use the following rules for severity: ...".
- "Given this file content and suggestion X, produce a minimal patch in unified diff format that applies to the current branch. Include tests or reasons if change may break behavior."

## Metrics to track
- Time from PR creation to auto-fix completion.
- % of review threads auto-fixed vs ignored vs escalated.
- Number of CI failures after auto-fix.
- Error rates and number of retries.
- Cost per iteration (USD).

## Open questions & choices to make
- Should the agent operate directly on the PR branch (simpler) or open an auto-fix child branch and push that as a follow-up commit into the PR? (Tradeoff: easier to track vs possible PR policy issues.)
- How aggressive should automatic fixes be for test-affecting or security-sensitive files?
- Single commit per PR iteration versus many small commits (affects ability to revert).
- Who resolves the review threads: the bot by calling GraphQL resolve mutation, or by posting a human-visible comment and letting GitHub UI status remain?

## Safety & audit recommendations
- Always leave a clear comment for every resolved thread explaining the decision.
- Tag auto-generated commits clearly (e.g., `auto-fix: <summary> (by continuous-claude)`).
- Persist a per-run audit file in the repo or remote store containing decisions and diffs.

## Example minimal lifecycle (concise)
1. Snapshot PR head + initial comment list.
2. Wait up to 20 minutes, poll for more comments.
3. Snapshot final comment list.
4. Generate decisions & patches.
5. Re-check PR head; if changed, re-run decision or merge incremental changes.
6. Apply patch, push, post thread comments and resolve threads.
7. Wait for CI; run final LLM check.
8. If OK, merge PR and delete branch.

---

If you'd like, I can next:
- Draft the specific changes to continuous_claude.sh (or a companion script) to implement the "wait-for-comments, collect -> decide -> patch -> resolve threads" flow, including the exact gh GraphQL queries/mutations and example Claude prompt templates.
- Or produce a prototype sequence of `gh` + `claude` commands for a single PR to validate the flow end-to-end.

Which next step do you want: a concrete patch plan for the current script, or a small standalone prototype implementation? 
