---
allowed-tools: Bash(git fetch*), Bash(git checkout*), Bash(git pull*), Bash(git log*), Bash(git diff*), Bash(git status*), Bash(git branch*), Bash(git ls-remote*), Bash(git add *), Bash(git commit*), Bash(git push*), Bash(git tag*), Bash(gh pr list*), Bash(gh pr view*), Bash(gh pr create*), Bash(npx tsc*), Bash(npx vue-tsc*), Bash(npm test*), Bash(npm run*), Bash(yarn*), Bash(pnpm*), Bash(mypy *), Bash(pyright *), Bash(pytest*), Bash(ruff*), Bash(go vet*), Bash(go test*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Edit, Grep, Glob
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

2. Verify `dev` branch exists on the remote (after fetching):
   ```bash
   git fetch origin
   git ls-remote --exit-code --heads origin dev
   ```
   If the command fails (exit code non-zero), stop:
   > "No `dev` branch found on the remote. This project may use a single-branch workflow — open a PR manually."

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
```

If not already on `dev`, switch to it — creating a local tracking branch if needed:
```bash
git checkout dev 2>/dev/null || git checkout -b dev --track origin/dev
git pull origin dev
```

If the pull fails, stop and report the error.

### Step 3 — Check what's being released

```bash
git log origin/main..dev --oneline
```

If empty, stop:
> "Nothing to release — dev and main are already in sync."

Show the commit list to the user.

### Step 4 — Verify (typecheck + test + lint)

Run the shared verification checks defined in `commands/_verify.md` against local `dev`. If any check fails, stop:
> "Verification failed on dev: <check>. Fix before releasing."

This is a safety net — CI is authoritative, but catching breakage locally avoids opening a release PR on a broken dev.

### Step 5 — Confirm and open the release PR

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

Show the user what's about to be created before calling `gh pr create`:

```
About to open release PR:
  Base:    main
  Head:    dev
  Title:   release: merge dev into main
  Commits: N (listed above)
  Body:
  ---
  <filled-body>
  ---
Proceed? [y/n]
```

Wait for approval. Only after the user confirms:

```bash
gh pr create \
  --base main \
  --head dev \
  --title "release: merge dev into main" \
  --body "<body>"
```

### Step 6 — Return to previous branch

```bash
git checkout -
```

### Step 7 — Suggest version bump

1. **Find the current version.** Search the project root for files containing a version field (look for `"version"` keys or `version =` patterns). Read the first match and extract the current semver value.

2. **Determine bump type** from the commits in Step 3:
   - Any commit message body or footer contains `BREAKING CHANGE` → **major**
   - Any commit type is `feat` → **minor**
   - Otherwise (all `fix`, `chore`, `docs`, `refactor`, etc.) → **patch**

3. **Compute the suggested next version** by incrementing the appropriate semver component (reset lower components to 0).

4. **Present the suggestion** — do not apply it automatically:
   > "Suggested version bump: `<current>` → `<suggested>` (based on N feat / M fix / K other commits). Apply? [y/n]"

5. If the user accepts, apply the version change to the file where it was found.

### Step 8 — Update changelog

1. **Check if a changelog file exists** (look for `CHANGELOG.md`, `CHANGELOG`, or similar in the project root). If none exists, skip this step.

2. **Group the commits from Step 3** by conventional commit type into sections:
   - `feat` → **Added**
   - `fix` → **Fixed**
   - `refactor` → **Changed**
   - `perf` → **Performance**
   - `docs` → **Documentation**
   - `chore`, `ci`, `build`, `style`, `test` → **Maintenance**

   Omit empty sections. Use the commit summary as the entry text. Include the scope if present.

3. **Add a new entry** at the top of the changelog (below any header) using the version from Step 6 and today's date:
   ```markdown
   ## [<version>] - YYYY-MM-DD

   ### Added
   - <feat commit summaries>

   ### Fixed
   - <fix commit summaries>
   ```

4. **Show the changelog entry** to the user for confirmation before writing it.

### Step 9 — Commit and push

If any changes were made (version bump, changelog), commit them to `dev` and push before the PR is merged:
```
chore: bump version to <version> and update changelog
```

### Step 10 — Confirm and tag the release

Tags on `main` are effectively permanent — deleting a pushed tag is noisy and can break downstream consumers. Confirm before tagging:

```
About to tag the release:
  Tag:     v<version>
  Commit:  <short-sha> <commit-subject>
  Remote:  origin
Proceed? [y/n]
```

Wait for approval. Only after the user confirms:

```bash
git tag -a v<version> -m "v<version>"
git push origin v<version>
```

### Final output

```
Release PR opened: <url>
Includes N commits from dev → main.
Tagged: v<version>
```
