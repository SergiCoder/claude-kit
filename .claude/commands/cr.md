---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Read, Edit, Write, Glob, Grep, Agent
description: Run multi-profile code review on the current branch
---

Review code changes on the current branch using specialized reviewer profiles.

## Arguments

- `$ARGUMENTS` — optional: comma-separated profile names (e.g., `security,performance`) or `--fix` flag

If no profiles specified, auto-detect from changed files.

## Current state

**Current branch:** !`git branch --show-current`

## Instructions

### Step 1 — Determine base branch and diff

Determine the base branch: check if `dev` branch exists — if yes use `dev`, otherwise use `main`.

Get the list of changed files:
```bash
git diff origin/<base>...HEAD --name-only
```

If the diff is empty, stop:
> "No changes compared to `<base>` — nothing to review."

### Step 2 — Select profiles

Available profiles are in `.claude/review/profiles/` (one `.md` file per profile).

**If profiles were specified in `$ARGUMENTS`:** run only those.

**If no profiles specified**, auto-detect from changed file paths:

| Changed files match | Profiles to run |
|---|---|
| `*.py` | security, language, performance, reuse, simplifier, testing |
| `*.go` | security, language, performance, reuse, simplifier, testing |
| `*.ts`, `*.tsx`, `*.js`, `*.jsx` | security, language, performance, reuse, simplifier, testing |
| `*.vue`, `*.svelte` | security, language, performance, simplifier |
| `Dockerfile*`, `docker-compose*`, `infra/**`, `.github/workflows/**` | security |
| `*.md`, `CLAUDE.md`, `README*` | documentation |
| `.env*` (committed only) | security |

Deduplicate. Always include `simplifier` if any source code changed. Always include `documentation` if any `.md` file changed.

If a `framework.md` profile exists in `.claude/review/profiles/`, include it when source code changes.

### Step 3 — Run reviews in parallel

For each selected profile, read `.claude/review/profiles/<profile>.md` and launch an Agent with `subagent_type: "general-purpose"` and `run_in_background: true`.

**For standard profiles** (all except `documentation` and `testing`):
```
You are reviewing code changes for a pull request.

## Profile Rules
<contents of .claude/review/profiles/<profile>.md>

## Project Context
Read CLAUDE.md (and any sub-CLAUDE.md files) for project rules and architecture decisions.

## Your Task
1. Get the diff: `git diff origin/<base>...HEAD`
2. Read the changed files relevant to your profile
3. For the REUSE profile: also search the broader codebase for existing implementations that overlap with new code
4. Apply every rule in your checklist against the changed code
5. Report findings using the output format in your profile rules
6. Skip rules with no violations — only report issues

Be specific: file paths, line numbers, concrete fix suggestions.
Do not report issues outside the diff unless the reuse profile requires it.
Group by severity: CRITICAL → HIGH → MEDIUM → LOW.
End with: X critical, Y high, Z medium, W low.
```

**For the `documentation` profile:**
```
You are a documentation reviewer that fixes issues directly.

## Profile Rules
<contents of .claude/review/profiles/documentation.md>

## Project Context
Read CLAUDE.md and any sub-CLAUDE.md files.

## Your Task
1. Get the diff: `git diff origin/<base>...HEAD`
2. Read all changed files and all documentation files
3. Check every rule in the profile
4. For each issue, fix it directly using the Edit tool
5. Summarize all fixes applied

You have permission to edit documentation files.
```

**For the `testing` profile:**
```
You are a testing reviewer that writes missing tests directly.

## Profile Rules
<contents of .claude/review/profiles/testing.md>

## Project Context
Read CLAUDE.md and any sub-CLAUDE.md files for testing patterns and conventions.

## Your Task
1. Get the diff: `git diff origin/<base>...HEAD`
2. Read changed source files to identify new/changed code that needs tests
3. Search for existing test coverage
4. Read existing test files to learn the project's testing style and fixtures
5. Write missing tests directly using Edit or Write tool
6. Summarize all tests written

You have permission to edit and create test files.
```

Launch ALL agents in parallel. Do NOT run sequentially.

### Step 4 — Collect and present results

Wait for all agents to complete, then present:

```markdown
# Code Review Report

**Branch:** <branch>
**Base:** <base>
**Files changed:** <count>
**Profiles run:** <list>

---

## Critical & High Findings
(All critical/high across profiles, grouped by file. Deduplicate — keep most detailed, note both profiles.)

## Medium Findings
(Same format)

## Low Findings
(Same format)

## Documentation Fixes Applied
(If documentation profile ran)

## Tests Written
(If testing profile ran)

## Summary

| Profile | Critical | High | Medium | Low |
|---|---|---|---|---|
| security | X | X | X | X |
| **Total (deduplicated)** | **X** | **X** | **X** | **X** |
```

### Step 5 — Fix mode (optional)

Determine which severities to fix based on the flag in `$ARGUMENTS`:

| Flag | Severities fixed |
|---|---|
| `--fix` | CRITICAL, HIGH |
| `--fix-medium` | CRITICAL, HIGH, MEDIUM |
| `--fix-all` | CRITICAL, HIGH, MEDIUM, LOW |

If any fix flag is present: for each finding in the applicable severities, ask "Fix this? [y/n/skip all]", apply with the Edit tool if yes, then re-run the relevant profile's checks on the fixed file to verify.

If no fix flag is present, do not modify any files.

### Final output

```
Review complete: X critical, Y high, Z medium, W low findings across N profiles.
```

- Critical findings: "CRITICAL findings must be resolved before opening a PR."
- High or below only: "HIGH findings should be resolved before merge. Run `/cr --fix` to address them."
- Clean: "No critical or high findings. Ready for PR."
