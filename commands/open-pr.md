---
allowed-tools: Bash(git fetch*), Bash(git rebase*), Bash(git diff*), Bash(git branch*), Bash(git push*), Bash(git log*), Bash(git status*), Bash(gh pr create*), Bash(gh pr view*), Bash(npx tsc*), Bash(npx vue-tsc*), Bash(npm test*), Bash(npm run*), Bash(yarn*), Bash(pnpm*), Bash(mypy *), Bash(pyright *), Bash(pytest*), Bash(ruff*), Bash(go vet*), Bash(go test*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read
description: Sync base branch, run relevant tests, open a PR
---

Open a pull request for the current branch.

## Current state

**Current branch:** !`git branch --show-current`

**Upstream tracking:** !`git status -sb 2>/dev/null | head -1`

## Instructions

### Step 1 — Determine target branch

Check if `dev` branch exists on origin. If yes, use `dev` as base. Otherwise use `main`.
Exception: if current branch starts with `hotfix/`, always target `main`.

### Step 2 — Sync with base branch

```bash
git fetch origin <base>
git rebase origin/<base>
```

If rebase fails with conflicts, stop:
> "Rebase conflict detected. Resolve the conflicts, then re-run /open-pr."

### Step 3 — Push

Check whether the rebase moved commits (i.e., HEAD differs from the pre-rebase remote tracking tip). If it did, the push will rewrite remote history — confirm with the user first:

```bash
git log @{u}..HEAD --oneline   # commits that would be force-pushed
git log HEAD..@{u} --oneline   # commits on remote that would be dropped
```

> "Rebase rewrote N local / dropped M remote commit(s). Force-push with lease? [y/n]"

If the branch is a clean fast-forward (no remote commits dropped), skip the prompt.

```bash
git push --force-with-lease
```

### Step 4 — Check the diff

```bash
git diff origin/<base>...HEAD --name-only
```

If empty, stop:
> "No changes compared to <base> — nothing to PR."

### Step 5 — Verify (typecheck + test + lint)

Run the shared verification checks defined in `commands/_verify.md`, scoping tests to changed areas when supported. If any check fails due to the branch's changes, stop:
> "Verification failed: <check>. Fix before opening a PR."

Note any pre-existing failures in the PR body under Testing.

### Step 6 — Build the PR description

Gather context:
```bash
git log origin/<base>...HEAD --oneline
git diff origin/<base>...HEAD --name-only
```

Build the PR body:

```markdown
## Summary
- <bullet points from commit messages — what changed and why>

## Type of change
- [ ] `feature/*` — new feature
- [ ] `fix/*` — bug fix
- [ ] `hotfix/*` — critical production fix
- [ ] `refactor` / `chore` / `docs`

(tick the one matching the branch prefix)

## Testing
<what was tested and result — e.g. "All tests pass">

## Checklist
- [ ] Code is self-reviewed
- [ ] Tests pass
- [ ] Lint passes
- [ ] No secrets committed
- [ ] PR targets the correct base branch
```

### Step 7 — Confirm and open the PR

Show the user exactly what's about to be created before calling `gh pr create`:

```
About to open PR:
  Base:   <base>
  Head:   <current-branch>
  Title:  <type>(<scope>): <summary>
  Body:
  ---
  <filled-body>
  ---
Proceed? [y/n]
```

Wait for approval. Only after the user confirms:

```bash
gh pr create \
  --base <base> \
  --title "<type>(<scope>): <summary>" \
  --body "<filled-body>"
```

Title: Conventional Commit format, ≤72 chars, imperative mood, lowercase, no period.

Use a HEREDOC for the body.

### Final output

```
PR opened: <url>
Title: <title>
Base: <base>
```
