# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-03-24

### Added
- Security checklist expanded with 13 new items: password hashing, tenant isolation, path traversal, unsafe deserialization, TLS enforcement, minimal containers, encryption at rest, audit trails, and new Cryptography and API Security sections
- Release command now suggests a semver version bump based on conventional commits and updates the changelog

### Changed
- Removed technology-specific references from agnostic skills and commands (security, quality, performance, testing, ship, open-pr, address-review) to keep them truly stack-agnostic

## [1.1.0] - 2026-03-24

### Added
- `address-review` command — fetches unresolved PR inline comments, validates each for false positives, fixes root causes, replies with explanations, resolves threads, and posts a summary table on the PR
- False-positive validation gate in `review` command — each profile agent now validates its own findings against surrounding code context, project conventions, actual reachability, and diff ownership before reporting

## [1.0.5] - 2026-03-24

### Added
- Self-review checklist item in `open-pr` PR body

### Changed
- CI review trigger changed to `@prism review` comment format
- Renamed `branch` command to `create-branch`

## [1.0.4] - 2026-03-23

### Added
- Expanded stack reviewer checklists with deeper coverage across all language/framework skills
- Installable CI review template and `install-ci-review` command

## [1.0.3] - 2026-03-22

### Fixed
- Resolve review threads via GraphQL after replying in `ship` command

## [1.0.2] - 2026-03-21

### Added
- Auto-resolve inline review comments after push in `ship` command

### Changed
- CI review switched to inline comments with deduplication and restricted trigger

## [1.0.1] - 2026-03-20

### Changed
- Simplified contribution model to single-maintainer fork+PR
- CI auto-approve job for maintainer PRs
- Review CI trigger simplified, added `@claude` comment trigger and allowedTools

## [1.0.0] - 2026-03-19

### Added
- Initial release as Claude Code plugin (`prism`)
- `review` command — multi-profile parallel code review with auto-detection of languages and frameworks
- `ship` command — pre-ship hygiene check, conventional commit, and push
- `open-pr` command — sync base branch, run tests, open PR with conventional commit title
- `create-branch` command — create feature/fix/hotfix branches from the correct base
- `release` command — open a release PR from dev into main
- Review profiles: quality, security, performance, documentation, testing
- Per-language stack skills: Python, TypeScript, Go
- Per-framework stack skills: Django, Flask, FastAPI, Next.js, Nuxt, SvelteKit, React, Vue, Express, Spring Boot, Laravel, ASP.NET Core, Rails, Go HTTP
- Profile ownership boundaries to prevent duplicate findings across profiles
- `--fix`, `--fix-medium`, `--fix-all` flags for automated fix mode in review
- GitHub Actions workflow for automated PR reviews
- Marketplace plugin support
