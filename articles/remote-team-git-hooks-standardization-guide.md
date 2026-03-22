---
layout: default
title: "Remote Team Git Hooks Standardization Guide"
description: "Standardize Git hooks across distributed teams using Husky, pre-commit, and Lefthook to enforce code quality at commit time"
date: 2026-03-22
author: theluckystrike
permalink: /remote-team-git-hooks-standardization-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Git hooks run on every developer's machine before commits are pushed. Without standardization, hooks exist in some team members' repos and not others, leading to inconsistent code quality, broken CI pipelines, and merge conflicts from formatting differences. Remote teams feel this more acutely — there's no over-the-shoulder "hey, you're missing the linter" moment.

This guide covers three hook managers for different stack profiles: Husky (JavaScript teams), pre-commit (polyglot teams), and Lefthook (performance-sensitive workflows).

---

## The Problem with .git/hooks

The `.git/hooks/` directory is not tracked by version control. Each developer must manually copy or symlink hooks. That never happens consistently. The solution is to version hook configuration in the repo itself, then auto-install on `git clone` or initial setup.

---

## Husky (JavaScript/Node Teams)

Husky is the standard for JavaScript projects. It integrates with npm's `prepare` lifecycle so hooks install automatically after `npm install`.

```bash
npm install --save-dev husky
npx husky init
```

This creates `.husky/` directory (tracked) and adds `"prepare": "husky"` to `package.json`.

Add a pre-commit hook:

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged
```

Configure `lint-staged` in `package.json` to only run tools on staged files (much faster than running on all files):

```json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss}": [
      "prettier --write"
    ],
    "*.{json,md,yaml,yml}": [
      "prettier --write"
    ]
  }
}
```

Add a commit-msg hook for conventional commits:

```bash
# Install commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```bash
# .husky/commit-msg
#!/bin/sh
npx --no -- commitlint --edit $1
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'chore', 'style', 'refactor', 'ci', 'test', 'perf', 'revert'],
    ],
    'subject-max-length': [2, 'always', 100],
    'scope-case': [2, 'always', 'lower-case'],
  },
};
```

Add a pre-push hook to run tests before pushing to main:

```bash
# .husky/pre-push
#!/bin/sh

BRANCH=$(git branch --show-current)
if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "staging" ]; then
  echo "Running full test suite before push to $BRANCH..."
  npm test -- --passWithNoTests
fi
```

---

## pre-commit (Polyglot Teams)

pre-commit is Python-based but works across any language. It downloads and manages hook tools in isolated environments, so hooks run identically regardless of what's installed locally.

Install:

```bash
pip install pre-commit
# or
brew install pre-commit
```

Define hooks in `.pre-commit-config.yaml`:

```yaml
# .pre-commit-config.yaml
repos:
  # General file checks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key
      - id: check-large-files
        args: ['--maxkb=1024']
      - id: mixed-line-ending
        args: ['--fix=lf']

  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # Go
  - repo: https://github.com/dnephin/pre-commit-golang
    rev: v0.5.1
    hooks:
      - id: go-fmt
      - id: go-vet
      - id: go-unit-tests

  # Terraform
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.90.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint

  # Secrets detection
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

  # Dockerfile
  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint-docker
```

Install for all developers:

```bash
pre-commit install
pre-commit install --hook-type commit-msg
```

Run manually against all files:

```bash
pre-commit run --all-files
```

Auto-update hooks monthly:

```bash
pre-commit autoupdate
```

Add to CI to enforce hooks even if a developer bypasses locally:

```yaml
# .github/workflows/pre-commit.yml
name: Pre-commit checks
on: [push, pull_request]
jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - uses: pre-commit/action@v3.0.1
```

---

## Lefthook (Fast, Multi-language)

Lefthook is a Go binary with no runtime dependencies. It's significantly faster than Husky or pre-commit because it runs hooks in parallel.

Install:

```bash
# macOS
brew install lefthook

# npm (for JS monorepos)
npm install --save-dev lefthook

# Direct download
curl -1sLf 'https://dl.cloudsmith.io/public/evilmartians/lefthook/setup.deb.sh' | sudo bash
apt install lefthook
```

Configure in `lefthook.yml`:

```yaml
# lefthook.yml
pre-commit:
  parallel: true
  commands:
    lint:
      glob: "*.{js,ts,jsx,tsx}"
      run: npx eslint --fix {staged_files}
      stage_fixed: true

    format:
      glob: "*.{js,ts,jsx,tsx,json,css,md}"
      run: npx prettier --write {staged_files}
      stage_fixed: true

    go-fmt:
      glob: "*.go"
      run: gofmt -w {staged_files}
      stage_fixed: true

    go-vet:
      glob: "*.go"
      run: go vet ./...

    secrets:
      run: detect-secrets scan --baseline .secrets.baseline {staged_files}

pre-push:
  commands:
    tests:
      run: go test ./...
    build:
      run: go build ./...

commit-msg:
  commands:
    commitlint:
      run: npx commitlint --edit {1}
```

Install hooks:

```bash
lefthook install
```

Lefthook runs `lint`, `format`, `go-fmt`, `go-vet`, and `secrets` checks in parallel. On a modern laptop this completes in under 2 seconds for most changesets.

Skip hooks when needed (rare, break-glass):

```bash
LEFTHOOK=0 git commit -m "wip: debug session"
```

---

## Distributing Hooks in a Monorepo

For monorepos with multiple packages, scope hooks to the affected package:

```yaml
# lefthook.yml (monorepo root)
pre-commit:
  commands:
    frontend-lint:
      root: "packages/frontend/"
      glob: "*.{ts,tsx}"
      run: npx eslint --fix {staged_files}
      stage_fixed: true

    backend-vet:
      root: "services/api/"
      glob: "*.go"
      run: go vet ./...

    infra-validate:
      root: "infra/terraform/"
      glob: "*.tf"
      run: terraform validate
```

---

## Enforce via Makefile

Create a `make setup` target so new team members get hooks with one command:

```makefile
# Makefile
.PHONY: setup hooks

setup: hooks
	@echo "Development environment ready"

hooks:
	@which pre-commit > /dev/null || pip install pre-commit
	pre-commit install
	pre-commit install --hook-type commit-msg
	@echo "Git hooks installed"

# Allow skipping hooks in CI (hooks run separately)
ci-check:
	pre-commit run --all-files
```

Document this in your onboarding runbook:

```
# New developer setup
git clone git@github.com:your-org/your-repo.git
cd your-repo
make setup
```

---

## Related Reading

- [Best Tools for Remote Team Code Ownership](/remote-work-tools/best-tools-remote-team-code-ownership/)
- [Remote Team Code Review Checklist Template](/remote-work-tools/remote-team-code-review-checklist-template/)
- [How to Create a Remote Dev Environment Template](/remote-work-tools/how-to-create-a-remote-dev-environment-template/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
