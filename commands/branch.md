---
allowed-tools: Bash(git fetch*), Bash(git checkout*), Bash(git pull*), Bash(git push*)
description: Create a feature, fix, or hotfix branch from the correct base
---
Create a new git branch using the project's branching strategy.

The user will pass arguments in the format: `<type> <name>`
- type: `feature`, `fix`, or `hotfix`
- name: branch name in kebab-case

Rules:
- `feature` and `fix` branch from `dev` (fallback to `main` if `dev` doesn't exist)
- `hotfix` branches from `main`
- Always fetch origin and pull the base branch before creating

Steps to execute:
1. Parse the type and name from $ARGUMENTS (e.g. "feature my-feature" or "hotfix critical-bug")
2. Validate type is one of: feature, fix, hotfix
3. Determine base branch: `hotfix` → `main`; `feature`/`fix` → `dev` if it exists, else `main`
4. Run these commands in order:
   - `git fetch origin`
   - `git checkout <base>` then `git pull origin <base>`
   - `git checkout -b <type>/<name>`
   - `git push -u origin <type>/<name>`
5. Confirm the branch was created and pushed, and remind the user where to PR into.

If type or name is missing, ask the user to provide them before proceeding.
