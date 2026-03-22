---
layout: default
title: "How to Set Up Renovate for Dependency Updates"
description: "Configure Renovate Bot to automate dependency updates across remote team repos with grouping, scheduling, and auto-merge for patch releases"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-renovate-dependency-updates/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Outdated dependencies are a security liability and a technical debt accumulation point. Renovate automates dependency updates by opening PRs, grouping related updates, and auto-merging safe patches — so remote teams get current without drowning in manual update work.

## Key Takeaways

- **Topics covered**: installation options, option 1: github app (easiest), option 2: self-hosted with github actions
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## Installation Options

### Option 1: GitHub App (Easiest)

```bash
# 1. Install the Mend Renovate app from GitHub Marketplace
# 2. Grant access to your repositories
# 3. Add renovate.json to your repo root
```

### Option 2: Self-Hosted with GitHub Actions

```yaml
# .github/workflows/renovate.yml
name: Renovate

on:
  schedule:
    - cron: '0 4 * * 1-5'  # Weekdays at 4am
  workflow_dispatch:

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: renovatebot/github-action@v40
        with:
          configurationFile: renovate.json
          token: ${{ secrets.RENOVATE_TOKEN }}
        env:
          LOG_LEVEL: debug
```

### Option 3: Self-Hosted CLI

```bash
# Install
npm install -g renovate

# Run against a repo
RENOVATE_TOKEN=your-github-token \
RENOVATE_PLATFORM=github \
renovate yourorg/yourrepo
```

## Base Configuration

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    ":disableDependencyDashboard"
  ],
  "timezone": "America/New_York",
  "schedule": ["before 6am on Monday"],
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2,
  "commitMessagePrefix": "chore(deps):",
  "labels": ["dependencies"],
  "assignees": ["@yourteam/backend"],
  "reviewers": ["@yourteam/leads"]
}
```

## Grouping Updates

Reduce PR noise by grouping related packages:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "packageRules": [
    {
      "groupName": "AWS SDK",
      "matchPackagePatterns": ["^@aws-sdk/", "^aws-sdk"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "React core",
      "matchPackageNames": ["react", "react-dom", "@types/react", "@types/react-dom"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "ESLint and plugins",
      "matchPackagePatterns": ["^eslint", "^@typescript-eslint/"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "Testing libraries",
      "matchPackagePatterns": ["^jest", "^@testing-library/", "^vitest"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "Storybook",
      "matchPackagePatterns": ["^@storybook/", "^storybook"],
      "matchUpdateTypes": ["minor", "patch"]
    },
    {
      "groupName": "Development tools",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ]
}
```

## Auto-Merge Safe Updates

```json
{
  "packageRules": [
    {
      "description": "Auto-merge patch updates for trusted packages",
      "matchPackagePatterns": ["^@types/"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "platformAutomerge": true
    },
    {
      "description": "Auto-merge patch updates for dev dependencies",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "description": "Never auto-merge major updates",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "dependencyDashboard": true
    },
    {
      "description": "Pin Docker digest updates (auto-merge)",
      "matchDatasources": ["docker"],
      "matchUpdateTypes": ["digest"],
      "automerge": true
    }
  ]
}
```

## Python / Poetry Configuration

```json
{
  "extends": ["config:base"],
  "pip_requirements": {
    "fileMatch": ["requirements.*\\.txt$"]
  },
  "poetry": {
    "fileMatch": ["pyproject\\.toml$"]
  },
  "packageRules": [
    {
      "matchManagers": ["poetry"],
      "matchDepTypes": ["dev-dependencies"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    },
    {
      "matchManagers": ["poetry"],
      "matchDepTypes": ["dependencies"],
      "matchUpdateTypes": ["major"],
      "labels": ["major-update", "review-required"]
    }
  ]
}
```

## Docker and GitHub Actions Updates

```json
{
  "extends": ["config:base"],
  "docker-compose": {
    "fileMatch": ["docker-compose.*\\.yml$"]
  },
  "dockerfile": {
    "fileMatch": ["(^|/)Dockerfile[^/]*$"]
  },
  "github-actions": {
    "fileMatch": ["\\.github/workflows/.*\\.yml$"]
  },
  "packageRules": [
    {
      "matchManagers": ["github-actions"],
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchManagers": ["dockerfile"],
      "matchUpdateTypes": ["patch"],
      "automerge": false
    },
    {
      "description": "Pin Docker base image digests",
      "matchManagers": ["dockerfile"],
      "pinDigests": true
    }
  ]
}
```

## Security-Only Mode

For repos where you only want to act on known vulnerabilities:

```json
{
  "extends": ["config:base", ":onlyNpmDependencies"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "assignees": ["@yourteam/security"],
    "automerge": false,
    "minimumReleaseAge": "0 days"
  },
  "osvVulnerabilityAlerts": true,
  "packageRules": [
    {
      "matchUpdateTypes": ["major", "minor", "patch"],
      "enabled": false
    },
    {
      "matchCategories": ["security"],
      "enabled": true
    }
  ]
}
```

## Monorepo Configuration

```json
{
  "extends": ["config:base"],
  "ignorePaths": [
    "**/node_modules/**",
    "**/vendor/**"
  ],
  "packageRules": [
    {
      "matchPaths": ["services/api/**"],
      "groupName": "api-dependencies",
      "schedule": ["before 6am on Tuesday"],
      "assignees": ["@yourteam/backend"]
    },
    {
      "matchPaths": ["services/web/**"],
      "groupName": "web-dependencies",
      "schedule": ["before 6am on Wednesday"],
      "assignees": ["@yourteam/frontend"]
    },
    {
      "matchPaths": ["infrastructure/**"],
      "matchManagers": ["terraform"],
      "groupName": "terraform-providers",
      "schedule": ["before 6am on Thursday"],
      "assignees": ["@yourteam/platform"]
    }
  ]
}
```

## PR Description Customization

```json
{
  "prBodyTemplate": "This PR contains the following updates:\n\n{{{table}}}\n\n{{{notes}}}\n\n{{{changelogs}}}\n\n---\n\n**Merge checklist:**\n- [ ] CI passes\n- [ ] No breaking changes in changelog\n- [ ] Tested in staging if applicable",
  "commitBody": "Renovate automated update — review the changelog above"
}
```

## Running Renovate on Gitea

```json
{
  "platform": "gitea",
  "endpoint": "https://git.example.com",
  "token": "your-gitea-token",
  "repositories": [
    "mycompany/api",
    "mycompany/web",
    "mycompany/infrastructure"
  ]
}
```

```bash
RENOVATE_PLATFORM=gitea \
RENOVATE_ENDPOINT=https://git.example.com \
RENOVATE_TOKEN=your-token \
renovate mycompany/api
```

## Related Reading

- [How to Automate Code Quality Gates for Remote Teams](/remote-work-tools/how-to-automate-code-quality-gates-remote-teams/)
- [How to Set Up Verdaccio Private npm Registry](/remote-work-tools/how-to-set-up-verdaccio-private-npm-registry/)
- [Best DevsSecOps Toolchain for Remote Teams](/remote-work-tools/best-devsecops-toolchain-for-remote-teams-integrating-securi/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
