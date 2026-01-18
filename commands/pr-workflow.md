---
description: NON-NEGOTIABLE PR workflow - must be followed for EVERY pull request
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - mcp__github__*
---

# PR Workflow

**This checklist MUST be followed for EVERY pull request. No exceptions.**

## Phase 1: Before Creating PR

Execute these validation steps before creating the PR:

### 1. Detect Project Type and Run Validations

```bash
# Determine project type and run appropriate checks
```

**For JavaScript/TypeScript projects:**
1. Run `pnpm typecheck` or `npm run typecheck` or `yarn typecheck`
2. Run `pnpm build` or `npm run build` or `yarn build`
3. Run `pnpm test` or `npm test` or `yarn test`

**For Python projects:**
1. Run `mypy .` or equivalent type checking
2. Run `python -m pytest` or equivalent test command
3. Run `ruff check .` or `flake8` for linting

**For Makefiles present:**
1. Check for `make typecheck`, `make build`, `make test` targets
2. Prefer make commands over direct package manager commands

### 2. Security Scan

Before committing, verify:
- [ ] No hardcoded paths (e.g., `/Users/username/...`)
- [ ] No secrets, API keys, or tokens
- [ ] No machine-specific configuration values
- [ ] Environment variables used for sensitive data

### 3. Create the PR

Use the GitHub MCP tools to:
1. Search for `pull_request_template.md` in the repository
2. Use the template structure for the PR description
3. Create the PR with `mcp__github__create_pull_request`

---

## Phase 2: Autonomous PR Monitoring (REQUIRED)

**You MUST autonomously monitor and manage PRs through to merge.**

### 4. Monitor Bot Reviews

Poll the PR status every 30-60 seconds until all bot reviews complete:

```
Use BOTH of these methods to get complete check status:

1. mcp__github__pull_request_read with method "get_status"
   → Returns commit statuses (Vercel, etc.)

2. mcp__github__pull_request_read with method "get"
   → Check the "mergeable_state" field in the response
```

**Understanding Check Types:**
- **Commit Statuses** (`get_status`): Older API, used by Vercel, some CI systems
- **Check Runs**: Newer API used by GitHub Actions, Cursor Bugbot, Copilot, Sentry
- **IMPORTANT**: `get_status` does NOT show Check Runs! You must also check `mergeable_state`.

**Mergeable State Values:**
- `"clean"` → All checks passed, safe to merge
- `"unstable"` → Some checks are failing or pending - DO NOT MERGE
- `"blocked"` → Branch protection rules are blocking merge
- `"behind"` → Branch needs to be updated with base branch
- `"unknown"` → GitHub is still computing - poll again

**Polling Strategy:**
1. Call `mcp__github__pull_request_read` with `method: "get_status"` to see commit statuses
2. Call `mcp__github__pull_request_read` with `method: "get"` and check `mergeable_state`
3. If `mergeable_state` is NOT `"clean"`, wait 30-60 seconds and poll again
4. **DO NOT proceed to merge unless `mergeable_state` is `"clean"`**
5. Continue polling until `mergeable_state` becomes `"clean"` or a final failure state

### 5. Address ALL Bot Feedback and Mark Resolved

When bot reviews complete:

1. **Read every comment:**
   - Use `mcp__github__pull_request_read` with `method: "get_review_comments"`
   - Use `mcp__github__pull_request_read` with `method: "get_comments"`
   - **Track the count of threads where `IsResolved: false`** - this is your unresolved count

2. **For each issue raised:**
   - Analyze the feedback
   - Implement the fix in the codebase
   - Commit and push the changes
   - Reply to the comment confirming the fix using `mcp__github__add_issue_comment`
   - **CRITICAL: Mark the thread as resolved** using the gh CLI (see below)

3. **DO NOT ignore any feedback** - every bot comment must be addressed

4. **Mark threads as resolved after addressing:**
   ```bash
   # Get the thread ID from the review comment response (the "ID" field starting with "PRRT_")
   # Then resolve it:
   gh api graphql -f query='
     mutation {
       resolveReviewThread(input: {threadId: "PRRT_xxxxx"}) {
         thread { isResolved }
       }
     }
   '
   ```
   
   **Why this matters:** Without marking threads resolved, you cannot reliably distinguish between addressed and unaddressed comments. This leads to premature merges with outstanding issues.

### 6. Wait for Re-validation

After pushing fixes:
1. Poll status checks again (same as Step 4)
2. All checks must show `SUCCESS` (not `NEUTRAL` or `IN_PROGRESS`)
3. **CRITICAL: After EACH poll cycle, re-fetch review comments**
   - Call `mcp__github__pull_request_read` with `method: "get_review_comments"`
   - Check for NEW comments that appeared since your last push
   - Bots generate new comments on each new commit - you MUST read them
4. If any check produces new feedback, repeat Step 5
5. **DO NOT assume previous fixes resolved all issues** - always re-check comments

**The Polling Loop Must Include Comment Checking AND Resolution Tracking:**
```
While mergeable_state != "clean" OR unresolved_thread_count > 0:
  1. Wait 30-60 seconds
  2. Call get_status and get to check status
  3. Call get_review_comments to check for NEW feedback ← CRITICAL
  4. Count threads where IsResolved == false ← THIS IS YOUR GATE
  5. If unresolved_thread_count > 0:
     a. Address each unresolved thread
     b. Push fixes
     c. Mark each addressed thread as resolved (gh api graphql)
  6. Repeat until BOTH conditions are met:
     - mergeable_state == "clean"
     - unresolved_thread_count == 0
```

**NEVER merge when unresolved_thread_count > 0, even if mergeable_state is "clean".**

---

## Phase 3: Merge Criteria

**ALL conditions must be TRUE before merging:**

### 7. Final Verification Checklist

**MANDATORY PRE-MERGE CHECKS (ALL must pass):**

```
1. Call mcp__github__pull_request_read with method "get" and verify:
   - mergeable_state == "clean"  ← GATE #1
   - merged == false (PR not already merged)

2. Call mcp__github__pull_request_read with method "get_review_comments" and verify:
   - Count threads where IsResolved == false
   - unresolved_thread_count == 0  ← GATE #2
```

**TWO GATES must pass before merge:**
1. **`mergeable_state` is `"clean"`** - If not, DO NOT MERGE
2. **`unresolved_thread_count` is `0`** - If not, DO NOT MERGE

**Both conditions must be TRUE. Either one failing blocks the merge.**

Additional requirements:
- [ ] `mergeable_state` is `"clean"` (not "unstable", "blocked", or "behind")
- [ ] **Zero unresolved review threads** (every `IsResolved` must be `true`)
- [ ] All commit statuses show `SUCCESS` (from `get_status`)
- [ ] No failing CI jobs or Check Runs
- [ ] Every addressed comment has been marked as resolved via GitHub API

### 8. Execute Merge

**Only after `mergeable_state` is `"clean"`:**

```bash
# Merge the PR
gh pr merge <PR_NUMBER> --merge

# Or use MCP tool:
# mcp__github__merge_pull_request with merge_method: "merge"
```

**NEVER merge when `mergeable_state` is `"unstable"` - this means checks are pending or failing.**

### 9. Verify Success

After merge:
1. Confirm the merge was successful
2. Report the final status to the user

---

## Error Handling

### If checks fail repeatedly:
1. After 3 failed attempts at the same issue, pause and ask for human guidance
2. Provide a summary of what was tried and what failed

### If bot feedback is unclear:
1. Ask clarifying questions before implementing a fix
2. Document the ambiguity in the PR comments

### If merge is blocked:
1. Check for branch protection rules
2. Check for required reviewers
3. Report the blocking condition to the user

---

## Common Mistakes to Avoid

### Mistake: Merging when `mergeable_state` is "unstable"
**Problem:** `get_status` only shows commit statuses, not GitHub Check Runs. You see "1 check passed" but there are actually 3 checks, 2 of which are pending.
**Fix:** ALWAYS check `mergeable_state` from the PR details. Only merge when it's `"clean"`.

### Mistake: Not actually waiting between polls
**Problem:** Calling status checks once and proceeding immediately.
**Fix:** If checks are pending, actually wait 30-60 seconds before polling again. Use multiple polling iterations.

### Mistake: Treating `mergeable: true` as approval to merge
**Problem:** `mergeable` only means there are no git conflicts. It does NOT mean checks passed.
**Fix:** Check `mergeable_state`, not just `mergeable`. They are different fields with different meanings:
  - `mergeable: true` → No git conflicts
  - `mergeable_state: "clean"` → All checks passed, safe to merge

### Mistake: Not re-checking comments after each push
**Problem:** After pushing a fix, bots run again and may generate NEW comments. Polling only `mergeable_state` and ignoring new comments leads to missed feedback.
**Fix:** After EVERY push, and during EVERY poll cycle, call `get_review_comments` to check for new unresolved threads. Address ALL unresolved threads before attempting to merge.

### Mistake: Only checking if threads are "resolved" vs reading content
**Problem:** The `IsResolved` field tells you if a thread is marked resolved, but new bots may have added entirely new threads that you haven't seen.
**Fix:** Track the number of review comments. If the count increases, read ALL threads and address any new or unresolved ones.

### Mistake: Not marking threads as resolved after addressing them
**Problem:** You address a comment by pushing a fix and replying, but don't mark the thread as resolved. Later, when checking if the PR is ready to merge, you can't distinguish between addressed and unaddressed comments. You assume everything is done and merge prematurely.
**Fix:** After addressing EVERY comment:
1. Push the fix
2. Reply confirming the fix
3. **Immediately mark the thread as resolved** using `gh api graphql` with `resolveReviewThread`
4. Before merging, verify `unresolved_thread_count == 0`

This creates a clean audit trail: resolved threads = addressed issues, unresolved threads = outstanding work.

### Mistake: Merging when `IsResolved: false` threads exist
**Problem:** You see `mergeable_state: "clean"` and assume the PR is ready. But there are still threads with `IsResolved: false` that you either forgot about or didn't notice.
**Fix:** Add an explicit gate: **Count threads where `IsResolved == false`**. If that count is > 0, DO NOT MERGE, regardless of `mergeable_state`. Address and resolve ALL threads first.

---

## Quick Reference Commands

```bash
# Check PR status
gh pr status

# View PR checks
gh pr checks <PR_NUMBER>

# View PR comments
gh pr view <PR_NUMBER> --comments

# Merge PR (only after BOTH gates pass)
gh pr merge <PR_NUMBER> --merge

# RESOLVE A REVIEW THREAD (required after addressing each comment)
gh api graphql -f query='
  mutation {
    resolveReviewThread(input: {threadId: "PRRT_xxxxx"}) {
      thread { isResolved }
    }
  }
'
```

**MCP Tools:**
- `mcp__github__create_pull_request` - Create PR
- `mcp__github__pull_request_read` - Get PR details, diff, status, comments
  - `method: "get"` → Returns `mergeable_state` (the true indicator of check status)
  - `method: "get_status"` → Returns commit statuses only (NOT Check Runs!)
  - `method: "get_review_comments"` → Returns code review comments with `IsResolved` field
  - `method: "get_comments"` → Returns PR conversation comments
- `mcp__github__merge_pull_request` - Merge PR (only when BOTH gates pass)
- `mcp__github__add_issue_comment` - Reply to comments

**Thread Resolution:**
- Get thread ID from `get_review_comments` response (field: `ID`, starts with `PRRT_`)
- Resolve using `gh api graphql` with `resolveReviewThread` mutation
- Verify resolution: re-fetch comments and confirm `IsResolved: true`
