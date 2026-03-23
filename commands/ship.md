---
allowed-tools: Bash(git diff*), Bash(git status*), Bash(git ls-files*), Bash(git branch*), Bash(git remote*), Bash(git add*), Bash(git commit*), Bash(git push*), Bash(git restore --staged*), Bash(test -f .git/MERGE_HEAD*), Bash(gh pr view*), Bash(gh api *), Read, Glob, Grep
description: Pre-ship hygiene check, conventional commit, and push
---
Analyze the current git repository state and create well-structured conventional commits, then push.

## Current repository state

**Merge status:**
!`test -f .git/MERGE_HEAD && echo "PENDING_MERGE=true" || echo "PENDING_MERGE=false"`

**Staged changes (diff):**
!`git diff --cached --stat 2>/dev/null || echo "none"`

**Staged diff content:**
!`git diff --cached 2>/dev/null || echo "none"`

**Unstaged tracked changes:**
!`git diff --stat 2>/dev/null || echo "none"`

**Unstaged diff content:**
!`git diff 2>/dev/null || echo "none"`

**Untracked files:**
!`git ls-files --others --exclude-standard 2>/dev/null || echo "none"`

**Current branch:**
!`git branch --show-current 2>/dev/null`

**Remote tracking:**
!`git remote -v 2>/dev/null | head -4 || echo "no remote"`

---

## Instructions

### Step 1 — Safety check
If `PENDING_MERGE=true` is shown above, **stop immediately** and tell the user:
> "A merge is in progress. Resolve the merge conflict first, then re-run /ship."

If staged changes, unstaged diff, and untracked files are all empty, **stop immediately**:
> "Nothing to commit — working tree is clean."

### Step 2 — Pre-commit hygiene check
Scan changed and untracked files for:
1. **Empty files** — 0-byte files are likely scaffolding leftovers. Suggest deleting.
2. **Orphan files** — not imported or referenced anywhere. Suggest deleting.
3. **Misplaced files** — location doesn't match project structure conventions.
4. **Duplicate files** — same name and similar content in different locations.

Only flag genuine issues. If issues found, list them and ask the user before proceeding.

### Step 3 — Auto-format (if applicable)
Check if the project has a linter/formatter configured (look for config files: `.ruff.toml`, `pyproject.toml`, `.eslintrc*`, `biome.json`, `golangci.yml`, etc.).

If a formatter is found and there are changed files in the relevant language, run it before staging. Only run formatters that are clearly configured — do not guess.

### Step 4 — Commit staged changes
If there are staged changes:
1. **Group by single purpose.** If staged changes span multiple logical units, split them.
2. **Write a Conventional Commit message:**
   ```
   <type>(<scope>): <short summary>

   [optional body — what changed and why, not how]
   ```
   Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`
   Scope: affected module, package, or layer
   Summary: imperative mood, lowercase, no period, ≤72 chars

3. Commit: `git commit -m "<message>"`

### Step 5 — Commit unstaged/untracked changes
Group by single purpose, stage explicitly by file name (never `git add .` or `git add -A`), write a Conventional Commit message, commit.

Skip: `.env*` files, build artifacts, auto-generated files, secrets.

### Step 6 — Push
```bash
git push
# or if no upstream:
git push -u origin <branch-name>
```

### Step 7 — Resolve inline review comments
After pushing, check if there is an open PR for the current branch:
```bash
gh pr view --json number,url 2>/dev/null
```

If a PR exists:
1. Get the list of files that were part of the commits just pushed (from Steps 4-5).
2. Fetch unresolved bot review threads and their comments:
   ```bash
   gh api graphql -f query='
   {
     repository(owner: "{owner}", name: "{repo}") {
       pullRequest(number: {number}) {
         reviewThreads(first: 50) {
           nodes {
             id
             isResolved
             comments(first: 1) {
               nodes { body path }
             }
           }
         }
       }
     }
   }' --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | {threadId: .id, path: .comments.nodes[0].path, body: .comments.nodes[0].body}'
   ```
3. Also fetch the REST comments to check for existing replies:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments \
     --jq '[.[] | select((.user.type == "Bot" or .user.login == "github-actions[bot]") and .in_reply_to_id == null) | {id, path, line, body}]'
   ```
4. For each unresolved thread where `path` matches one of the files just committed:
   - Read the comment body to understand what issue was flagged.
   - Check the current state of the file to confirm the issue was addressed by the commits.
   - If fixed:
     a. If there is no reply yet, reply with a short, friendly message explaining what was done. Be specific about the fix — reference the actual change, not just "fixed". Vary your tone naturally (don't repeat the same phrase across replies).
     b. Resolve the thread using GraphQL:
        ```bash
        gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "{threadId}"}) { thread { isResolved } } }'
        ```
   - If NOT fixed (the code still has the issue), skip it silently.
5. If no PR exists or no comments match, skip this step silently.

### Final output
```
Commit 1: feat(auth): add JWT refresh token rotation
Commit 2: fix(api): handle empty response from upstream
Pushed to origin/<branch>
Resolved 3 review comments on PR #42
```
