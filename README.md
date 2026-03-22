# claude-kit

Portable Claude Code toolkit — custom commands and review profiles designed as a starting point for any repo. Drop the `.claude/` folder into a project and it works.

## Structure

```
.claude/
├── commands/          # Slash commands
│   ├── branch.md      # Create feature/fix/hotfix branches
│   ├── commit.md      # Conventional commits with hygiene checks
│   ├── cr.md          # Multi-profile parallel code review
│   ├── pr.md          # Open a pull request
│   └── release.md     # Open a release PR (dev → main)
└── review/
    └── profiles/      # Reviewer profiles used by /cr
        ├── documentation.md
        ├── performance.md
        ├── quality.md
        ├── security.md
        ├── stack.md
        └── testing.md
```

## Commands

| Command | Description |
|---|---|
| `/branch <type> <name>` | Create a `feature/`, `fix/`, or `hotfix/` branch from the correct base |
| `/commit` | Pre-commit hygiene check, auto-format, conventional commits, push |
| `/cr [profiles] [--fix\|--fix-medium\|--fix-all]` | Run multi-profile code review in parallel |
| `/pr` | Sync base branch, run tests, open a PR |
| `/release` | Open a release PR from `dev` into `main` |

## Code Review

`/cr` auto-detects which profiles to run based on changed file paths. You can also specify profiles explicitly:

```bash
/cr                        # auto-detect profiles
/cr security,performance   # specific profiles only
/cr --fix                  # review + interactively fix CRITICAL and HIGH
/cr --fix-medium           # review + fix CRITICAL, HIGH, and MEDIUM
/cr --fix-all              # review + fix all severities
```

### Review Profiles

| Profile | What it checks |
|---|---|
| `security` | Auth, injection, secrets, headers, rate limiting, data exposure |
| `quality` | Dead code, duplication, stdlib/dependency reinvention, verbose patterns, size thresholds |
| `stack` | Language idioms, type safety, error handling — Python, TypeScript, Go + framework best practices (Django, Flask, FastAPI, Next.js, Nuxt, SvelteKit, Go) |
| `performance` | N+1 queries, indexes, async correctness, caching, frontend rendering |
| `documentation` | Fixes docs directly — CLAUDE.md, README, docstrings, inline comments |
| `testing` | Writes missing tests directly — happy path, error cases, edge cases |

## Stack Support

Commands auto-detect the stack from project config files (no configuration needed):
- **Test runners**: pytest, go test, npm/pnpm/yarn test, make test
- **Linters/formatters**: Ruff, ESLint, Biome, golangci-lint, and others
- **Base branch**: uses `dev` if it exists, falls back to `main`

## GitHub PR Reviews (Automated)

The workflow in `.github/workflows/claude-review.yml` runs the same review profiles automatically on every pull request and posts findings as a single PR comment.

### Setup

1. Copy `.claude/` and `.github/` into your repo
2. Add your Anthropic API key as a repository secret named `ANTHROPIC_API_KEY`
   - GitHub repo → Settings → Secrets and variables → Actions → New repository secret
3. Open a PR — the review runs automatically

The workflow auto-detects which profiles to run based on changed files, same as `/cr` locally.

## Usage

Copy `.claude/` into any project. Also copy `.github/` if you want automated PR reviews.

For project-specific overrides, edit the profiles directly — the files are designed to be modified per repo.
