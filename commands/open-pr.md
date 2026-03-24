---
allowed-tools: Bash(git fetch*), Bash(git rebase*), Bash(git diff*), Bash(git branch*), Bash(git push*), Bash(git log*), Bash(git status*), Bash(gh pr create*), Bash(gh pr view*), Read
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

```bash
git push --force-with-lease
```

### Step 4 — Check the diff

```bash
git diff origin/<base>...HEAD --name-only
```

If empty, stop:
> "No changes compared to <base> — nothing to PR."

### Step 5 — Run relevant tests

Look at the changed file paths and detect the test runner from project config:
- Check dependency manifests, test config files, and `Makefile` for a test target
- Use the project's configured test command

Run only the test suites for areas that have changed files. Tell the user which suite you're running before running it. If tests fail, stop:
> "Tests failed. Fix the failures before opening a PR."

If no test runner is detectable, skip this step and note it in the PR.

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

### Step 7 — Open the PR

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
