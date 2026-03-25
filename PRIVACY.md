# Privacy Policy

**Last updated:** 2026-03-25

## Overview

Prism is a Claude Code plugin that runs entirely within your local Claude Code session. It does not collect, store, transmit, or process any personal data or telemetry.

## Data Collection

Prism does **not** collect any data. Specifically:

- No analytics or tracking
- No cookies or local storage
- No network requests to external servers
- No user accounts or authentication

## How It Works

All commands run locally inside your Claude Code environment. Code review, commits, branching, and PR operations are performed through your existing Git and GitHub CLI configuration. The only network activity is what Git and the GitHub CLI perform on your behalf (e.g., `git push`, `gh pr create`).

When using the optional CI review workflow, the GitHub Actions runner communicates directly with the Anthropic API using your own API key stored as a repository secret. Prism does not intermediate, proxy, or log these requests.

## Third-Party Services

Prism itself does not interact with any third-party services. The tools it invokes (Git, GitHub CLI, Anthropic API) are governed by their own privacy policies:

- [GitHub Privacy Statement](https://docs.github.com/en/site-policy/privacy-policies/github-general-privacy-statement)
- [Anthropic Privacy Policy](https://www.anthropic.com/privacy)

## Contact

If you have questions about this policy, open an issue at [github.com/SergiCoder/prism](https://github.com/SergiCoder/prism).
