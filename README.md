# prism

Multi-profile code review, conventional commits, branching, and PR workflow — packaged as a Claude Code plugin for any repo.

## Commands

| Command | Description |
|---|---|
| `/prism:branch <type> <name>` | Create a `feature/`, `fix/`, or `hotfix/` branch from the correct base |
| `/prism:commit` | Pre-commit hygiene check, auto-format, conventional commits, push |
| `/prism:cr [profiles] [--fix\|--fix-medium\|--fix-all]` | Run multi-profile code review in parallel |
| `/prism:pr` | Sync base branch, run tests, open a PR |
| `/prism:release` | Open a release PR from `dev` into `main` |

## Code Review

`/prism:cr` auto-detects which profiles to run based on changed file paths. You can also specify profiles explicitly:

```bash
/prism:cr                        # auto-detect profiles
/prism:cr security,performance   # specific profiles only
/prism:cr --fix                  # review + interactively fix CRITICAL and HIGH
/prism:cr --fix-medium           # review + fix CRITICAL, HIGH, and MEDIUM
/prism:cr --fix-all              # review + fix all severities
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

## Plugin Structure

```
prism/
├── .claude-plugin/
│   └── plugin.json          # plugin manifest
├── commands/                # user-invoked slash commands
│   ├── branch.md
│   ├── commit.md
│   ├── cr.md
│   ├── pr.md
│   └── release.md
└── skills/                  # model-invoked review profiles
    ├── security/SKILL.md
    ├── quality/SKILL.md
    ├── stack/SKILL.md
    ├── performance/SKILL.md
    ├── documentation/SKILL.md
    └── testing/SKILL.md
```

## GitHub PR Reviews (Automated)

The workflow in `.github/workflows/claude-review.yml` runs the same review profiles automatically on every pull request and posts findings as a single PR comment.

### Setup

1. Install the plugin: `/plugin install SergiCoder/prism`
2. Add your Anthropic API key as a repository secret named `ANTHROPIC_API_KEY`
   - GitHub repo → Settings → Secrets and variables → Actions → New repository secret
3. Copy `.github/workflows/claude-review.yml` into your repo
4. Open a PR — the review runs automatically
