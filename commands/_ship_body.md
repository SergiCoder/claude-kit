# Ship procedure (shared)

Shared body for `/ship` and `/ship-fast`. Analyze the current git repository state and create well-structured conventional commits, then push.

## Current repository state

**Merge status:**
!`test -f .git/MERGE_HEAD && echo "PENDING_MERGE=true" || echo "PENDING_MERGE=false"`

**Staged changes (stat):**
!`git diff --cached --stat 2>/dev/null || echo "none"`

**Unstaged tracked changes (stat):**
!`git diff --stat 2>/dev/null || echo "none"`

**Untracked files:**
!`git ls-files --others --exclude-standard 2>/dev/null || echo "none"`

**Current branch:**
!`git branch --show-current 2>/dev/null`

**Upstream:**
!`git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "no upstream"`

Full diff content is NOT inlined. Fetch on demand with `git diff --cached <path>` or `git diff <path>` only when needed to write a commit message.

---

## Instructions

### Step 1 — Safety check
If `PENDING_MERGE=true` is shown above, **stop immediately** and tell the user:
> "A merge is in progress. Resolve the merge conflict first, then re-run /ship."

If staged changes, unstaged diff, and untracked files are all empty, **stop immediately**:
> "Nothing to commit — working tree is clean."

### Step 2 — Auto-format (if applicable)
Check if the project has a linter/formatter configured (look for formatter/linter config files in the project root).

If a formatter is found and there are changed files in the relevant language, run it before staging. Only run formatters that are clearly configured — do not guess.

After formatting, re-stage any files that were already staged and got reformatted (`git add <file>` for each). This ensures formatting changes are included in the commit.

### Step 3 — Verify (typecheck + test + lint)

Run the shared verification checks defined in `commands/_verify.md`. **Fail fast here** — do not proceed to the hygiene scan or commit-message generation if verify fails on the current edits.

If a check fails and the failure is caused by the changes about to be shipped, stop:
> "Verification failed: <check>. Fix before shipping."

If a check is pre-existing on the base branch, note it and continue.

### Step 4 — Pre-commit hygiene check
Scan only for **clear, obvious** issues without speculative file reads:
1. **Empty files** — 0-byte files are likely scaffolding leftovers. Suggest deleting.
2. **Obvious scaffolding leftovers** visible from filenames or untracked list (e.g. `untitled.txt`, `foo.bak`, editor swap files).

Do not perform broad orphan/duplicate searches across the tree — those rarely pay off and burn tokens. If issues are found, list them and ask the user before proceeding.

### Step 5 — Commit staged changes
If there are staged changes:
1. **Group by single purpose.** If staged changes span multiple logical units, split them. Use `git diff --cached <path>` to inspect content only when grouping requires it.
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
