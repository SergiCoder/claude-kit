---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(git push*), Bash(git add *), Bash(git commit*), Bash(gh pr view*), Bash(gh api *), Read, Edit, Glob, Grep, Agent
description: Address inline PR review comments — validate, fix root causes, reply, and resolve
---

Address unresolved inline review comments on the current branch's PR.

## Current state

**Current branch:** !`git branch --show-current`

## Instructions

### Step 1 — Find the PR and fetch unresolved comments

```bash
gh pr view --json number,url,headRefName 2>/dev/null
```

If no PR exists for the current branch, stop:
> "No open PR found for this branch."

Extract owner/repo from the remote:
```bash
git remote get-url origin
```

Fetch all unresolved review threads with their full comment chains:
```bash
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          path
          line
          comments(first: 10) {
            nodes { body author { login } }
          }
        }
      }
    }
  }
}' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false)'
```

If no unresolved threads, stop:
> "No unresolved review comments — nothing to address."

### Step 2 — Validate and fix each comment

For each unresolved thread, process sequentially:

#### 2a — Deep false-positive validation

Before writing any code, determine if the comment raises a real issue:

1. **Read the full file** — not just the flagged line. Understand the surrounding context: callers, error handling boundaries, types, tests.
2. **Trace the concern** — if the comment flags missing validation, check callers and middleware. If it flags an error path, check if it's handled upstream. If it flags a pattern, check if it's an established project convention (CLAUDE.md, existing code).
3. **Check actual reachability** — is the issue reachable in practice? Could the type system, framework guarantees, or upstream logic prevent it?
4. **Check diff ownership** — is the flagged code actually changed in this PR, or pre-existing?

**Verdict:**
- If the concern is already handled elsewhere, unreachable, contradicts project conventions, or targets pre-existing code → mark as **false positive**.
- Otherwise → mark as **real issue**.

#### 2b — Fix real issues at root cause

For issues marked as real:

1. **Identify the root cause** — don't patch symptoms. If the comment flags a missing null check, ask why the null is possible in the first place. If it flags a race condition, fix the concurrency design, not just the symptom.
2. **Apply the fix** using the Edit tool. The fix must:
   - Address the structural cause, not paper over it
   - Be consistent with the surrounding code style and project conventions
   - Not introduce new issues
3. **Verify the fix** — re-read the changed code to confirm correctness.

#### 2c — Reply to the comment

Compose a reply based on the verdict:

**If false positive** — explain concisely why the issue doesn't apply. Be specific: name the caller that handles it, the type that prevents it, or the convention that justifies it. Keep it respectful — the reviewer raised a reasonable concern.

**If real issue, now fixed** — explain what the root cause was and what you changed. Reference the actual fix (e.g., "Added validation in `parseInput()` since the caller doesn't guarantee non-null"), not just "fixed".

Post the reply. Find the first comment's `id` in the thread via REST:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --jq '[.[] | select(.path == "{path}") | {id, body, line}]'
```

Then reply:
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  -X POST \
  -f body="{reply}" \
  -F in_reply_to={comment_id}
```

#### 2d — Resolve the conversation

Only resolve if:
- The issue was a validated false positive (with explanation posted), OR
- The issue was real AND the fix has been applied and verified

```bash
gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "{threadId}"}) { thread { isResolved } } }'
```

Do NOT resolve if the issue is real but you couldn't fix it — leave it open and note why in your reply.

### Step 3 — Commit and push fixes

If any files were modified:

1. Stage only the changed files by name (never `git add .`).
2. Commit with a conventional commit message summarizing the fixes:
   ```
   fix(<scope>): address review feedback

   <brief list of what was fixed and why>
   ```
3. Push:
   ```bash
   git push
   ```

If no files were modified (all comments were false positives), skip this step.

### Step 4 — Post summary comment on the PR

Post a summary comment on the PR so reviewers see what was addressed:

```bash
gh pr comment {number} --body "{summary}"
```

Use this format:

```markdown
## Review feedback addressed

| # | File | Comment | Verdict | Action |
|---|------|---------|---------|--------|
| 1 | `path/to/file.py:42` | <short description of concern> | Fixed | <what was changed> |
| 2 | `path/to/other.ts:17` | <short description of concern> | False positive | <why it doesn't apply> |
| 3 | `path/to/api.go:88` | <short description of concern> | Skipped | <why it couldn't be fixed> |

**Result:** X fixed, Y false positives resolved, Z left open
```

Use a HEREDOC to pass the body.

### Final output

```
Addressed N review comments on PR #<number>:
  - X fixed (root cause addressed)
  - Y false positives (explained and resolved)
  - Z skipped (could not fix — left open)
Pushed <M> commits to origin/<branch>
```
