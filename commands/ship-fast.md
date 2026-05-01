---
allowed-tools: Bash(git diff*), Bash(git status*), Bash(git ls-files*), Bash(git branch*), Bash(git rev-parse*), Bash(git add*), Bash(git commit*), Bash(git push*), Bash(git restore --staged*), Bash(test -f .git/MERGE_HEAD*), Bash(npx tsc*), Bash(npx vue-tsc*), Bash(npm test*), Bash(npm run*), Bash(yarn*), Bash(pnpm*), Bash(mypy *), Bash(pyright *), Bash(pytest*), Bash(ruff*), Bash(go vet *), Bash(go test*), Bash(bundle exec*), Bash(vendor/bin/phpunit*), Bash(dotnet build*), Bash(dotnet test*), Read, Glob, Grep
description: Pre-ship hygiene check, conventional commit, and push — runs on Haiku for lower cost on small changes
model: claude-haiku-4-5-20251001
---

@commands/_ship_body.md
