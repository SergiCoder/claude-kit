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

`/prism:cr` auto-detects the stack from config files and runs only the relevant reviewers. You can also specify profiles explicitly:

```bash
/prism:cr                        # auto-detect stack and profiles
/prism:cr security,performance   # specific profiles only
/prism:cr --fix                  # review + interactively fix CRITICAL and HIGH
/prism:cr --fix-medium           # review + fix CRITICAL, HIGH, and MEDIUM
/prism:cr --fix-all              # review + fix all severities
```

### Core Profiles

| Profile | What it checks |
|---|---|
| `security` | Auth, injection, secrets, headers, rate limiting, data exposure |
| `quality` | Dead code, duplication, stdlib/dependency reinvention, verbose patterns, size thresholds |
| `performance` | N+1 queries, indexes, async correctness, caching, frontend rendering |
| `documentation` | Fixes docs directly — CLAUDE.md, README, docstrings, inline comments |
| `testing` | Writes missing tests directly — happy path, error cases, edge cases |

### Stack Profiles (auto-detected)

| Profile | Language / Framework |
|---|---|
| `stack-python` | Python idioms, type safety, error handling |
| `stack-typescript` | TypeScript/JS idioms, type safety, error handling |
| `stack-go` | Go idioms, concurrency, type safety |
| `stack-django` | Models, queries, views, settings, migrations |
| `stack-flask` | Structure, blueprints, error handling |
| `stack-fastapi` | Routes, dependencies, Pydantic v2 |
| `stack-react` | Hooks, state, memoization, accessibility |
| `stack-vue` | Composition API, reactivity, Pinia |
| `stack-nextjs` | App Router, server/client components, performance |
| `stack-nuxt` | Composables, SSR correctness |
| `stack-sveltekit` | Svelte 5 runes, data loading, form actions |
| `stack-express` | Middleware, async error handling, input validation |
| `stack-spring` | Layers, DI, JPA, error handling, configuration |
| `stack-laravel` | Eloquent, controllers, validation, migrations |
| `stack-aspnet` | Controllers, DI lifetimes, EF Core, configuration |
| `stack-rails` | MVC, ActiveRecord, security, migrations |
| `stack-go-http` | Handlers, middleware, error responses |

Stack detection is automatic from: `package.json`, `pyproject.toml`, `requirements.txt`, `go.mod`, `pom.xml`, `build.gradle`, `composer.json`, `Gemfile`, `*.csproj`.

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
    ├── performance/SKILL.md
    ├── documentation/SKILL.md
    ├── testing/SKILL.md
    ├── stack-python/SKILL.md
    ├── stack-typescript/SKILL.md
    ├── stack-go/SKILL.md
    ├── stack-django/SKILL.md
    ├── stack-flask/SKILL.md
    ├── stack-fastapi/SKILL.md
    ├── stack-react/SKILL.md
    ├── stack-vue/SKILL.md
    ├── stack-nextjs/SKILL.md
    ├── stack-nuxt/SKILL.md
    ├── stack-sveltekit/SKILL.md
    ├── stack-express/SKILL.md
    ├── stack-spring/SKILL.md
    ├── stack-laravel/SKILL.md
    ├── stack-aspnet/SKILL.md
    ├── stack-rails/SKILL.md
    └── stack-go-http/SKILL.md
```

## GitHub PR Reviews (Automated)

The workflow in `.github/workflows/claude-review.yml` runs the same review profiles automatically on every pull request and posts findings as a single PR comment.

### Setup

1. Install the plugin: `/plugin install SergiCoder/prism`
2. Add your Anthropic API key as a repository secret named `ANTHROPIC_API_KEY`
   - GitHub repo → Settings → Secrets and variables → Actions → New repository secret
3. Copy `.github/workflows/claude-review.yml` into your repo
4. Open a PR — the review runs automatically
