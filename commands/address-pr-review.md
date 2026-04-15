---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(git push*), Bash(git add *), Bash(git commit*), Bash(gh pr view*), Bash(gh pr comment*), Bash(gh api *), Bash(git remote*), Bash(npx tsc*), Bash(npx vue-tsc*), Bash(npm test*), Bash(npm run*), Bash(yarn*), Bash(pnpm*), Bash(mypy *), Bash(pyright *), Bash(pytest*), Bash(ruff*), Bash(go vet*), Bash(go test*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Edit, Glob, Grep, Agent
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

1. **State the root cause** before editing. Write down (for use in the reply in 2c):
   - **Structural cause:** what design problem allows the issue to occur (e.g., "input is trusted without validation because the boundary check lives two layers up and doesn't cover this path")
   - **Why a naive patch is insufficient:** what the symptom-level fix would be and why it would recur or mask the real problem
   - **Correct design:** what the structural fix is

   **Exemption:** skip this statement for trivial mechanical changes where root-cause framing is overkill — typos, import reorders, renames, formatting, one-line nits. Only apply the exemption when the reviewer's concern is clearly Low severity AND the fix is a one-line mechanical change. When in doubt, write the statement.

2. **Apply the fix** using the Edit tool. The fix must:
   - Address the structural cause identified above, not paper over it
   - Be consistent with the surrounding code style and project conventions
   - Not introduce new issues
3. **Verify the fix** — re-read the changed code to confirm it matches the "correct design" you stated.

#### 2c — Reply to the comment

Compose a reply based on the verdict:

**If false positive** — explain concisely why the issue doesn't apply. Be specific: name the caller that handles it, the type that prevents it, or the convention that justifies it. Keep it respectful — the reviewer raised a reasonable concern.

**If real issue, now fixed** — carry forward the root-cause statement from 2b. The reply must name the structural cause and the fix, not just "fixed". Example: "Root cause: `parseInput()` trusted its caller, but the new webhook path bypasses the upstream validator. Fix: moved validation into `parseInput()` itself so every caller is covered."

For the trivial-nit exemption from 2b, a one-line acknowledgement is fine ("Fixed — typo in the error message").

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

### Step 3 — Verify fixes

If any files were modified, run the shared verification checks defined in `commands/_verify.md` before committing. If a check fails due to a fix, repair it and re-run before proceeding.

If no files were modified, skip this step.

### Step 4 — Commit and push fixes

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

### Step 5 — Post summary comment on the PR

Post a summary comment on the PR so reviewers see what was addressed:

```bash
gh pr comment {number} --body "{summary}"
```

Use this format:

```markdown
## Review feedback addressed

| Verdict | Critical | High | Medium | Low | Count |
|---|---|---|---|---|---|
| Fixed | X | X | X | X | X |
| False positive | X | X | X | X | X |
| Skipped | X | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** | **X** |

### Details

| # | File | Severity | Comment | Verdict | Action |
|---|------|----------|---------|---------|--------|
| 1 | `path/to/file:42` | High | <short description of concern> | Fixed | <what was changed> |
| 2 | `path/to/other:17` | Low | <short description of concern> | False positive | <why it doesn't apply> |
| 3 | `path/to/handler:88` | Medium | <short description of concern> | Skipped | <why it couldn't be fixed> |
```

Infer severity from the reviewer's framing (e.g. security/data-loss concerns → Critical/High; style/nits → Low). If unclear, default to Medium.

Use a HEREDOC to pass the body.

### Final output

```
Addressed N review comments on PR #<number>:
  - X fixed (root cause addressed)
  - Y false positives (explained and resolved)
  - Z skipped (could not fix — left open)
Pushed <M> commits to origin/<branch>
```
