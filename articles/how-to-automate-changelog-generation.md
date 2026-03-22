---
layout: default
title: "How to Automate Changelog Generation"
description: "Automate changelog generation from conventional commits using git-cliff, release-please, and GitHub Actions for consistent release documentation"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-changelog-generation/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manually writing changelogs means they either don't get written or they're vague summaries that don't help users understand what changed. Automated changelogs generated from structured commit messages give you release notes that are accurate, consistent, and require zero manual work.

The prerequisite is conventional commits. Without structured commit messages, automated tools can't categorize changes.

---

## git-cliff: Highly Configurable

git-cliff is a Rust-based changelog generator that reads conventional commits and produces Markdown. It's the most flexible option — you control exactly what appears and how it's formatted.

Install:

```bash
# macOS
brew install git-cliff

# Linux
cargo install git-cliff

# Docker
docker run -v "$(pwd)":/app orhunp/git-cliff:latest
```

Initialize configuration:

```bash
git cliff --init
```

This creates `cliff.toml`. A production-ready config:

```toml
# cliff.toml
[changelog]
header = "# Changelog\n\nAll notable changes to this project are documented in this file.\n"
body = """
{% for group, commits in commits | group_by(attribute="group") %}
## {{ group | striptags | trim | upper_first }}
{% for commit in commits %}
- {% if commit.scope %}**{{ commit.scope }}**: {% endif %}\
  {{ commit.message | upper_first }} ([{{ commit.id | truncate(length=7, end="") }}](https://github.com/your-org/your-repo/commit/{{ commit.id }}))\
  {% if commit.breaking %} [**BREAKING**]{% endif %}
{% endfor %}
{% endfor %}\n
"""
footer = ""
trim = true

[git]
conventional_commits = true
filter_unconventional = true
commit_parsers = [
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
  { message = "^perf", group = "Performance" },
  { message = "^docs", group = "Documentation" },
  { message = "^refactor", group = "Refactoring" },
  { message = "^ci", group = "CI/CD" },
  { message = "^chore\\(deps\\)", group = "Dependencies" },
  { message = "^chore", skip = true },
  { message = "^style", skip = true },
  { message = "^test", skip = true },
]
protect_breaking_commits = true
filter_commits = true
tag_pattern = "v[0-9]*"
skip_tags = "v0\\.1\\..*"
sort_commits = "newest"
```

Generate changelog for the current version:

```bash
# From last tag to HEAD
git cliff --output CHANGELOG.md

# For a specific version
git cliff --tag v1.2.0 --output CHANGELOG.md

# Only changes since last tag (for release notes)
git cliff --unreleased --strip header
```

---

## GitHub Actions: Automated Release on Tag

Trigger changelog generation and GitHub Release creation on every version tag:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history needed for git-cliff

      - name: Generate changelog
        id: changelog
        uses: orhun/git-cliff-action@v3
        with:
          config: cliff.toml
          args: --verbose --current --strip header
        env:
          OUTPUT: CHANGELOG.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body: ${{ steps.changelog.outputs.content }}
          draft: false
          prerelease: ${{ contains(github.ref_name, '-rc') || contains(github.ref_name, '-beta') }}
```

---

## release-please: Automated PRs

release-please (by Google) takes a different approach — it opens a "release PR" that accumulates changes until you're ready to release, then merges it to create the tag and changelog automatically.

```yaml
# .github/workflows/release-please.yml
name: Release Please

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: node  # or go, python, rust, simple
```

Configure which commit types go in the changelog:

```json
// .release-please-manifest.json
{
  ".": "1.0.0"
}
```

```json
// release-please-config.json
{
  "packages": {
    ".": {
      "release-type": "node",
      "changelog-sections": [
        {"type": "feat", "section": "Features"},
        {"type": "fix", "section": "Bug Fixes"},
        {"type": "perf", "section": "Performance"},
        {"type": "deps", "section": "Dependencies"},
        {"type": "revert", "section": "Reverts"}
      ],
      "bump-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    }
  }
}
```

release-please works well for library/package maintainers. For application deploys, git-cliff with manual tagging gives more control.

---

## conventional-changelog-cli (npm Ecosystem)

For JavaScript projects already using npm, `conventional-changelog-cli` integrates into your existing workflow:

```bash
npm install --save-dev conventional-changelog-cli
```

Add to `package.json`:

```json
{
  "scripts": {
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s",
    "version": "npm run changelog && git add CHANGELOG.md"
  }
}
```

The `version` script runs automatically when you call `npm version`:

```bash
# Bump patch version (fix commits)
npm version patch

# Bump minor version (feat commits)
npm version minor

# Bump major version (breaking changes)
npm version major
```

This updates `package.json`, generates `CHANGELOG.md`, commits both, and creates a git tag — all in one command.

---

## Enforcing Conventional Commits

Automated changelogs require conventional commit format. Enforce it:

```bash
# Install commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
};
```

```yaml
# .github/workflows/commitlint.yml
name: Commitlint
on:
  pull_request:
    types: [opened, synchronize, reopened, edited]

jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Lint commit messages
        uses: wagoid/commitlint-github-action@v6
        with:
          configFile: commitlint.config.js
          failOnWarnings: false
          helpURL: https://www.conventionalcommits.org
```

This runs on every PR and blocks merge if any commit message doesn't match the convention.

---

## Keep CHANGELOG.md Always Current

For projects where the changelog should be continuously updated (not just at release time), generate it in CI on every merge to main:

```yaml
# .github/workflows/update-changelog.yml
name: Update Changelog

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install git-cliff
        run: cargo install git-cliff

      - name: Update CHANGELOG.md
        run: git cliff --output CHANGELOG.md

      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add CHANGELOG.md
          git diff --cached --quiet || git commit -m "chore: update changelog [skip ci]"
          git push
```

The `[skip ci]` tag prevents the changelog update commit from triggering another CI run.

---

## Related Reading

- [Remote Team Git Hooks Standardization Guide](/remote-work-tools/remote-team-git-hooks-standardization-guide/)
- [How to Set Up Woodpecker CI for Self-Hosted](/remote-work-tools/how-to-set-up-woodpecker-ci-for-self-hosted/)
- [How to Create Automated Canary Deployments](/remote-work-tools/how-to-create-automated-canary-deployments/)

- [Automate Invoice Generation for Freelancers](/remote-work-tools/automate-invoice-generation-freelancers/)
---

## Related Articles

- [Best Changelog Tools for Remote Product Teams](/remote-work-tools/best-changelog-tools-for-remote-product-teams/)
- [Example celebration message generator (Python)](/remote-work-tools/how-to-write-remote-team-celebration-messages-that-acknowledge-effort-authentically-guide/)
- [Automate Invoice Generation for Freelancers](/remote-work-tools/automate-invoice-generation-freelancers/)
- [How to Write Effective Async Messages for Remote Work](/remote-work-tools/how-to-write-effective-async-messages-remote-work/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
