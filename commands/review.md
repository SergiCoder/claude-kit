---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(cat*), Bash(grep*), Read, Edit, Write, Glob, Grep, Agent
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

### Step 2 — Detect stack

Read the following files if they exist to detect languages and frameworks in use:
- `package.json` — for JS/TS framework detection
- `pyproject.toml` or `requirements.txt` — for Python framework detection
- `go.mod` — for Go module and framework detection
- `pom.xml` or `build.gradle` — for Java/Spring Boot detection
- `composer.json` — for PHP/Laravel detection
- `*.csproj` — for C#/ASP.NET Core detection
- `Gemfile` — for Ruby/Rails detection

**Language detection** (from changed file extensions):

| Changed files include | Languages detected |
|---|---|
| `*.py` | python |
| `*.ts`, `*.tsx`, `*.js`, `*.jsx`, `*.vue`, `*.svelte` | typescript |
| `*.go` | go |
| `*.java` | java |
| `*.php` | php |
| `*.cs` | csharp |
| `*.rb` | ruby |

**Framework detection** (from config files):

| Signal | Framework skill |
|---|---|
| `package.json` deps contain `"next"` | stack-nextjs |
| `package.json` deps contain `"nuxt"` | stack-nuxt |
| `package.json` deps contain `"@sveltejs/kit"` | stack-sveltekit |
| `package.json` deps contain `"react"` but NOT next/nuxt/sveltekit | stack-react |
| `package.json` deps contain `"vue"` but NOT nuxt | stack-vue |
| `package.json` deps contain `"express"` | stack-express |
| `pyproject.toml` or `requirements.txt` contains `django` | stack-django |
| `pyproject.toml` or `requirements.txt` contains `flask` | stack-flask |
| `pyproject.toml` or `requirements.txt` contains `fastapi` | stack-fastapi |
| `go.mod` contains `chi`, `gin`, `echo`, or changed files import `net/http` | stack-go-http |
| `pom.xml` or `build.gradle` contains `spring-boot` | stack-spring |
| `composer.json` contains `laravel/framework` | stack-laravel |
| Any `*.csproj` file exists in the repo | stack-aspnet |
| `Gemfile` contains `rails` | stack-rails |

Build the list of stack skills to run: one per detected language + one per detected framework. A framework skill covers framework-specific rules but does NOT replace the base language skill (e.g., `stack-django` + `stack-python` both run).

### Step 3 — Select profiles

**If profiles were specified in `$ARGUMENTS`:** run only those (skip auto-detection).

**If no profiles specified**, select profiles based on changed files and detected stack:

| Condition | Profiles to run |
|---|---|
| Any source code changed | quality + security + performance + testing + all detected stack skills |
| Any `*.py` | stack-python + any detected Python framework skill |
| Any `*.ts`, `*.tsx`, `*.js`, `*.jsx` | stack-typescript + any detected JS framework skill |
| Any `*.go` | stack-go + stack-go-http (if detected) |
| Any `*.java` | stack-spring (if detected) |
| Any `*.php` | stack-laravel (if detected) |
| Any `*.cs` | stack-aspnet |
| Any `*.rb` | stack-rails (if detected) |
| Any `*.vue` | stack-vue + stack-typescript |
| Any `*.svelte` | stack-typescript |
| Any `*.md`, `CLAUDE.md`, `README*` | documentation |
| `Dockerfile*`, `docker-compose*`, `infra/**`, `.github/workflows/**` | security only |
| `.env*` (committed only) | security only |

Deduplicate. Always include `quality` if any source code changed. Always include `documentation` if any `.md` file changed.

#### Profile ownership boundaries

Each profile ONLY reports findings within its domain. If a finding could belong to multiple profiles, it belongs to the FIRST matching profile in this list:

1. **quality**: DRY violations, dead code, complexity, stdlib reinvention, over-abstraction, size thresholds
2. **stack-***: Language idioms, type safety, error handling conventions, framework-specific patterns
3. **security**: Authentication, authorization, injection, secrets, headers, rate limiting
4. **performance**: Query patterns, caching, async correctness, indexes, frontend rendering
5. **documentation**: Doc accuracy, staleness, completeness
6. **testing**: Test coverage gaps

### Step 4 — Run reviews in parallel

For each selected profile, read `skills/<profile>/SKILL.md` and launch an Agent with `subagent_type: "general-purpose"` and `run_in_background: true`.

**For standard profiles** (all except `documentation` and `testing`):
```
You are reviewing code changes for a pull request.

## Profile Rules
<contents of skills/<profile>/SKILL.md>

## Project Context
Read CLAUDE.md (and any sub-CLAUDE.md files) for project rules and architecture decisions.

## Your Task
1. Get the diff: `git diff origin/<base>...HEAD`
2. Read the changed files relevant to your profile
3. For the QUALITY profile: also search the broader codebase for existing implementations using the Search Protocol defined in the profile
4. Process every rule in your checklist **in document order**. For each rule:
   a. State the rule
   b. Identify relevant code in the diff
   c. Determine PASS or FINDING with a one-line justification
   Rules with no relevant code in the diff: PASS (not applicable).
5. After processing all rules, compile findings into the output format defined in your profile
6. Only report findings — do not include PASS results in the final output

## Concreteness Gate
Before reporting any finding, verify it meets ALL of these criteria:
- You can point to a specific line or range in the diff
- You can name the exact rule from the checklist it violates
- The fix is a concrete code change, not a design opinion
If any criterion fails, do not report the finding.

## Root-Cause Requirement
Each finding must include a root-cause explanation: what structural design problem causes this issue, and what the correct design would be. Do not report symptoms (e.g., "this function is too long") without the underlying cause (e.g., "this function mixes I/O with business logic — extract business logic into a pure function").

## Ownership Boundary
Only report findings within your profile's domain as defined in the Profile Rules above.
If a finding crosses domains, defer to the profile listed first in the ownership boundary list.

Be specific: file paths, line numbers, concrete fix suggestions.
Do not report issues outside the diff unless the quality profile's search protocol requires it.
Group by severity: CRITICAL → HIGH → MEDIUM → LOW.
End with: X critical, Y high, Z medium, W low.
```

**For the `documentation` profile:**
```
You are a documentation reviewer that fixes issues directly.

## Profile Rules
<contents of skills/documentation/SKILL.md>

## Project Context
Read CLAUDE.md and any sub-CLAUDE.md files.

## Your Task
1. Get the diff: `git diff origin/<base>...HEAD`
2. Read all changed files and all documentation files
3. Process every rule in document order — check each one systematically
4. For each issue, fix it directly using the Edit tool
5. Summarize all fixes applied

You have permission to edit documentation files.
```

**For the `testing` profile:**
```
You are a testing reviewer that writes missing tests directly.

## Profile Rules
<contents of skills/testing/SKILL.md>

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

### Step 5 — Collect and present results

Wait for all agents to complete, then present:

```markdown
# Code Review Report

**Branch:** <branch>
**Base:** <base>
**Files changed:** <count>
**Stack detected:** <languages and frameworks>
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

### Step 6 — Fix mode (optional)

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
- High or below only: "HIGH findings should be resolved before merge. Run `/prism:review --fix` to address them."
- Clean: "No critical or high findings. Ready for PR."
