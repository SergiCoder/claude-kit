---
allowed-tools: Bash(git diff*), Bash(git status*), Bash(git ls-files*), Bash(git branch*), Bash(git remote*), Bash(git add*), Bash(git commit*), Bash(git push*), Bash(git restore --staged*), Bash(test -f .git/MERGE_HEAD*), Bash(npx tsc*), Bash(npx vue-tsc*), Bash(npm test*), Bash(npm run*), Bash(yarn*), Bash(pnpm*), Bash(mypy *), Bash(pyright *), Bash(pytest*), Bash(ruff*), Bash(go vet *), Bash(go test*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Glob, Grep
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
Check if the project has a linter/formatter configured (look for formatter/linter config files in the project root).

If a formatter is found and there are changed files in the relevant language, run it before staging. Only run formatters that are clearly configured — do not guess.

After formatting, re-stage any files that were already staged and got reformatted (`git add <file>` for each). This ensures formatting changes are included in the commit.

### Step 4 — Verify (typecheck + test + lint)

Run the shared verification checks defined in `commands/_verify.md`. If any check fails and the failure is caused by the changes about to be shipped, stop:
> "Verification failed: <check>. Fix before shipping."

If a check is pre-existing on the base branch, note it and continue.

### Step 5 — Commit staged changes
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

### Step 6 — Commit unstaged/untracked changes
Group by single purpose, stage explicitly by file name (never `git add .` or `git add -A`), write a Conventional Commit message, commit.

Skip: `.env*` files, build artifacts, auto-generated files, secrets.

### Step 7 — Push
```bash
git push
# or if no upstream:
git push -u origin <branch-name>
```

### Final output
```
Commit 1: feat(auth): add JWT refresh token rotation
Commit 2: fix(api): handle empty response from upstream
Pushed to origin/<branch>
```
