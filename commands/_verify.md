# Verification checks (shared)

Shared post-action verification logic used by ship, open-pr, release, address-pr-review, and review-and-fix.

Run the checks that are **configured in the project** — never fabricate commands. Detect from project config:

## Detection

| Signal | Typecheck | Test | Lint |
|---|---|---|---|
| `tsconfig.json` | `npx tsc --noEmit` | — | — |
| `package.json` deps contain `"vue-tsc"` | `npx vue-tsc --noEmit` | — | — |
| `package.json` script `test` | — | `npm test` (or `yarn test` / `pnpm test` per lockfile) | — |
| `package.json` script `lint` | — | — | `npm run lint` |
| `pyproject.toml` contains `mypy` or `[tool.mypy]` | `mypy .` | — | — |
| `pyrightconfig.json` or `pyproject.toml` contains `pyright` | `pyright` | — | — |
| `pyproject.toml` / `pytest.ini` configures pytest | — | `pytest` | — |
| `pyproject.toml` / `.ruff.toml` configures ruff | — | — | `ruff check .` |
| `go.mod` | `go vet ./...` | `go test ./...` | — |
| `Gemfile` contains `rspec` | — | `bundle exec rspec` | — |
| `Gemfile` contains `rubocop` | — | — | `bundle exec rubocop` |
| `composer.json` configures phpunit | — | `vendor/bin/phpunit` | — |
| `*.csproj` | `dotnet build` | `dotnet test` | — |
| `Makefile` has `test` / `lint` / `typecheck` target | per target | per target | per target |

Only run commands whose config file actually exists. If nothing is detected for a category, skip that category silently.

## Execution

- Run independent checks in parallel.
- Scope to changed areas when the tool supports it (e.g., test paths); otherwise run the full suite.
- Tell the user which checks you're about to run.

## On failure

1. Read the failure output.
2. Decide: caused by the current action's edits, or pre-existing on the base branch?
3. If caused by edits: fix the edit, then re-run only the failing check.
4. If pre-existing: note it and continue — do not block on it.

## Reporting

Record which checks ran and their pass/fail status so the calling command can include them in its final output.
