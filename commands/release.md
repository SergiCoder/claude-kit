---
allowed-tools: Bash(git fetch*), Bash(git checkout*), Bash(git pull*), Bash(git log*), Bash(git diff*), Bash(git status*), Bash(git branch*), Bash(gh pr list*), Bash(gh pr view*), Bash(gh pr create*), Read
description: Open a release PR from dev into main for production deploy
---

Open a pull request from `dev` into `main` for a production release.

## Current state

**Current branch:** !`git branch --show-current`

## Instructions

### Step 1 — Pre-flight checks

1. Ensure working tree is clean:
   ```bash
   git status --porcelain
   ```
   If there are uncommitted changes, stop:
   > "Working tree is not clean. Commit or stash your changes first, then re-run /release."

2. Verify `dev` branch exists:
   ```bash
   git branch -r | grep origin/dev
   ```
   If not found, stop:
   > "No `dev` branch found. This project may use a single-branch workflow — open a PR manually."

3. Check for open PRs targeting `dev` that haven't been merged:
   ```bash
   gh pr list --base dev --state open --json number,title,author
   ```
   If any exist, warn:
   > "There are N open PRs targeting dev that haven't been merged. Releasing now will exclude them. Continue? [y/n]"
   Wait for confirmation.

### Step 2 — Sync dev

```bash
git fetch origin
git checkout dev && git pull origin dev
```

If the pull fails, stop and report the error.

### Step 3 — Check what's being released

```bash
git log origin/main..dev --oneline
```

If empty, stop:
> "Nothing to release — dev and main are already in sync."

Show the commit list to the user.

### Step 4 — Open the release PR

```bash
gh pr create \
  --base main \
  --head dev \
  --title "release: merge dev into main" \
  --body "<body>"
```

Body:
```markdown
## Release

Merging `dev` into `main` for production deployment.

## Commits included

<each commit as a bullet from git log>

## Checklist

- [ ] CI passes
- [ ] No open PRs targeting dev that should be included
- [ ] Changelog reviewed (if applicable)
```

### Step 5 — Return to previous branch

```bash
git checkout -
```

### Final output

```
Release PR opened: <url>
Includes N commits from dev → main.
```
