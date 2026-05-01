---
allowed-tools: Bash(mkdir*), Bash(cp*), Bash(cat*), Bash(git status*), Read, Write, Glob
description: Install the Prism CI review workflow into the current project
model: claude-sonnet-4-6
---
Install the Prism automated PR review workflow into the current project.

## Instructions

1. Check that `.github/workflows/claude-review.yml` does not already exist. If it does, ask the user:
   > "A Claude review workflow already exists at `.github/workflows/claude-review.yml`. Overwrite it? [y/n]"
   Stop if they say no.

2. Create `.github/workflows/` directory if it doesn't exist.

3. Find the prism plugin directory by looking for the install template:
   - Check `~/.claude/plugins/SergiCoder/prism/install/claude-review.yml`
   - If not found, check common plugin paths or ask the user for the prism plugin path.

4. Copy the template:
   ```bash
   cp <prism-plugin-path>/install/claude-review.yml .github/workflows/claude-review.yml
   ```

5. Confirm to the user:
   > Installed `.github/workflows/claude-review.yml`
   >
   > To activate, add `ANTHROPIC_API_KEY` as a repository secret:
   > GitHub repo → Settings → Secrets and variables → Actions → New repository secret
   >
   > The workflow will run automatically on PRs and `@claude` mentions.
