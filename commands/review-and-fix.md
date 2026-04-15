---
allowed-tools: Bash(git diff*), Bash(git log*), Bash(git branch*), Bash(git status*), Bash(cat*), Bash(grep*), Bash(npm outdated*), Bash(npm audit*), Bash(npm view*), Bash(npm install*), Bash(npm update*), Bash(yarn outdated*), Bash(yarn audit*), Bash(yarn upgrade*), Bash(pnpm outdated*), Bash(pnpm audit*), Bash(pnpm update*), Bash(pip list*), Bash(pip-audit*), Bash(pip index*), Bash(pip install*), Bash(go list*), Bash(go get*), Bash(go version*), Bash(bundle outdated*), Bash(bundle update*), Bash(composer outdated*), Bash(composer update*), Bash(mvn versions*), Bash(dotnet list*), Bash(dotnet add*), Bash(node --version*), Bash(python --version*), Bash(python3 --version*), Bash(ruby --version*), Bash(php --version*), Bash(java --version*), Bash(dotnet --version*), Bash(npm run*), Bash(npm test*), Bash(yarn*), Bash(pnpm*), Bash(npx tsc*), Bash(tsc*), Bash(pytest*), Bash(mypy*), Bash(ruff*), Bash(go build*), Bash(go test*), Bash(go vet*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Edit, Write, Glob, Grep, Agent
description: Review code changes and fix all findings automatically
---

Review code changes on the current branch using specialized reviewer profiles, then automatically fix all findings.

## Arguments

- `$ARGUMENTS` — optional: comma-separated profile names (e.g., `security,performance`)

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
| Any backend framework detected (django, flask, fastapi, express, spring, laravel, rails, aspnet, go-http) | rest-api |
| Any `*.py` | stack-python + any detected Python framework skill |
| Any `*.ts`, `*.tsx`, `*.js`, `*.jsx` | stack-typescript + any detected JS framework skill |
| Any `*.go` | stack-go + stack-go-http (if detected) |
| Any `*.java` | stack-spring (if detected) |
| Any `*.php` | stack-laravel (if detected) |
| Any `*.cs` | stack-aspnet |
| Any `*.rb` | stack-rails (if detected) |
| Any `*.vue` | stack-vue + stack-typescript |
| Any `*.svelte` | stack-typescript |
| `Dockerfile*`, `docker-compose*`, `infra/**`, `.github/workflows/**` | security only |
| `.env*` (committed only) | security only |
| `package.json`, `pyproject.toml`, `requirements*.txt`, `go.mod`, `Gemfile`, `composer.json`, `*.csproj`, `pom.xml`, `build.gradle`, `build.gradle.kts` changed | dependencies |

Deduplicate. Always include `quality` if any source code changed. Always include `documentation` if any source code changed.

#### Profile ownership boundaries

Each profile ONLY reports findings within its domain. If a finding could belong to multiple profiles, it belongs to the FIRST matching profile in this list:

1. **quality**: DRY violations, dead code, complexity, stdlib reinvention, over-abstraction, size thresholds
2. **stack-***: Language idioms, type safety, error handling conventions, framework-specific patterns
3. **rest-api**: HTTP status codes, Location headers, error envelopes, resource representation, URL design, method semantics
4. **security**: Authentication, authorization, injection, secrets, headers, rate limiting
5. **performance**: Query patterns, caching, async correctness, indexes, frontend rendering
6. **documentation**: Doc accuracy, staleness, completeness
7. **testing**: Test coverage gaps
8. **dependencies**: Outdated packages, frameworks, and runtimes (writing profile — upgrades automatically)

### Step 4 — Run reviews in parallel

For each selected profile, launch an Agent with `subagent_type: "general-purpose"` and `run_in_background: true` using this prompt:

```
You are reviewing code changes for a pull request.

## Profile
Read and follow `skills/<profile>/SKILL.md`. It defines your scope, rules, behavior, and output format.

## Context
- Read CLAUDE.md (and any sub-CLAUDE.md files) for project rules and architecture decisions.
- Get the diff: `git diff origin/<base>...HEAD`
- Read changed files relevant to your profile's scope.

## Behavior Mode
**All profiles fix issues directly.** After identifying findings, apply fixes using the Edit and Write tools. Do not just report — fix.

- **Writing profiles** (testing, documentation, dependencies): follow your SKILL.md behavior to write tests, fix docs, or upgrade packages directly.
- **Reporting profiles** (all others): identify findings following the Review Standards and False-Positive Validation below, then **fix each finding** using the Edit tool. Summarize what you fixed.

**CI / read-only context:** If you do not have access to Edit or Write tools (e.g., running in CI), fall back to inline comment and reporting mode regardless of your profile. Post what you would have written as findings with file paths, suggested content, and severity, using the same inline comment flow as reporting profiles.

## Review Standards (reporting profiles only)
Before reporting any finding, verify ALL of these:
- You can point to a specific line or range in the diff
- You can name the exact rule from the checklist it violates
- The fix is a concrete code change, not a design opinion
If any criterion fails, do not report the finding.

Each finding must include a root-cause explanation: what structural design problem causes the issue and what the correct design would be.

## False-Positive Validation (reporting profiles only)
Before including any finding in your output, validate it is a real issue:

1. **Code context** — Read the file and surrounding lines (not just the diff hunk). Does the broader context already resolve the concern? (e.g., validation in a caller, a test covering the case, a comment explaining intent)
2. **Project conventions** — Check CLAUDE.md rules and existing patterns. Is the flagged code consistent with established project conventions?
3. **Actual impact** — Is the issue reachable in practice, or is it blocked by type system guarantees, framework behavior, or upstream validation?
4. **Diff ownership** — Is the flagged code actually changed in this PR, or is it pre-existing context?

**Drop** the finding if:
- Surrounding code already handles the concern
- It contradicts project conventions documented in CLAUDE.md
- The flagged code is not changed in the diff (pre-existing)
- The recommendation is speculative ("consider", "might want to") rather than a concrete rule violation

Only report findings that survive this validation.

## Ownership Boundary
Only report findings within your profile's domain.
If a finding crosses domains, defer to the profile listed first in the ownership boundary list.
```

Launch ALL agents in parallel. Do NOT run sequentially.

### Step 5 — Verify nothing broke

After all agents finish applying fixes, run the shared verification checks defined in `commands/_verify.md` to confirm the edits didn't introduce regressions. Record which checks ran and their pass/fail status for the final report.

### Step 6 — Collect and present results

Wait for all agents to complete, then present:

```markdown
# Review & Fix Report

**Branch:** <branch>
**Base:** <base>
**Files changed:** <count>
**Stack detected:** <languages and frameworks>
**Profiles run:** <list>
**Verification:** <which checks ran, pass/fail>

---

## Fixes Applied
(All fixes across all profiles, grouped by file. For each fix: severity, profile, what was wrong, what was changed.)

## Tests Written
(If testing profile ran — list tests added)

## Documentation Fixes Applied
(If documentation profile ran — list files updated)

## Dependencies Updated
(If dependencies profile ran — list packages upgraded with old → new versions)

## Summary

| Profile | Critical | High | Medium | Low | Fixed | Skipped (CI-only) |
|---|---|---|---|---|---|---|
| security | X | X | X | X | X | X |
| quality | X | X | X | X | X | X |
| **Total** | **X** | **X** | **X** | **X** | **X** | **X** |

**Do NOT include writing profiles (testing, documentation, dependencies) in the summary table.** Show their results only in their dedicated sections above.
```

### Step 7 — Final output

```
Review & fix complete: X findings fixed, Y tests written, Z docs updated, W packages upgraded across N profiles.
```

- If all findings were fixed and verification passed: "All findings resolved and checks passing. Ready for PR."
- If verification failed: list the failing checks and say "Fixes applied but verification failed — review before opening PR."
- If some findings could not be auto-fixed: list them with explanation and say "These require manual intervention."
