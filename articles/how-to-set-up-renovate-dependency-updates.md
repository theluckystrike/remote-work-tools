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

## Table of Contents

- [Prerequisites](#prerequisites)
- [Dependency Dashboard and Visibility](#dependency-dashboard-and-visibility)
- [Stabilization Period and Release Age](#stabilization-period-and-release-age)
- [Ignoring and Pinning Specific Packages](#ignoring-and-pinning-specific-packages)
- [Debugging and Testing Renovate Config](#debugging-and-testing-renovate-config)
- [Renovate with GitLab CI](#renovate-with-gitlab-ci)
- [Using Presets for Cross-Repo Consistency](#using-presets-for-cross-repo-consistency)
- [Measuring Renovate's Impact](#measuring-renovates-impact)
- [Related Reading](#related-reading)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install ation Options

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

### Step 2: Base Configuration

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

### Step 3: Grouping Updates

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

### Step 4: Auto-Merge Safe Updates

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

### Step 5: Python / Poetry Configuration

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

### Step 6: Docker and GitHub Actions Updates

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

### Step 7: Security-Only Mode

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

### Step 8: Monorepo Configuration

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

### Step 9: PR Description Customization

```json
{
  "prBodyTemplate": "This PR contains the following updates:\n\n{{{table}}}\n\n{{{notes}}}\n\n{{{changelogs}}}\n\n---\n\n**Merge checklist:**\n- [ ] CI passes\n- [ ] No breaking changes in changelog\n- [ ] Tested in staging if applicable",
  "commitBody": "Renovate automated update — review the changelog above"
}
```

### Step 10: Run Renovate on Gitea

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

## Dependency Dashboard and Visibility

The Dependency Dashboard is a GitHub issue Renovate keeps updated with a full picture of pending and blocked updates across your repo. Enable it selectively:

```json
{
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Renovate Dependency Dashboard",
  "dependencyDashboardHeader": "This dashboard lists all pending, blocked, and rate-limited updates.",
  "dependencyDashboardAutoclose": true,
  "packageRules": [
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboard": true,
      "dependencyDashboardApproval": true
    }
  ]
}
```

With `dependencyDashboardApproval: true`, Renovate will only open a PR for a major update after a team member checks the corresponding checkbox in the dashboard issue. This gives you a lightweight approval gate without a full review workflow.

## Stabilization Period and Release Age

Avoid being the first team to hit a broken release by requiring packages to age before Renovate acts on them:

```json
{
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "minimumReleaseAge": "3 days",
      "internalChecksFilter": "strict"
    },
    {
      "matchDepTypes": ["devDependencies"],
      "minimumReleaseAge": "1 day"
    },
    {
      "matchUpdateTypes": ["major"],
      "minimumReleaseAge": "7 days"
    }
  ]
}
```

The `internalChecksFilter: "strict"` setting tells Renovate to wait for its own internal age check to pass before opening the PR, even if the package satisfies other rules. Three days is a reasonable default that filters out yanked or immediately hotfixed releases without adding meaningful delay.

## Ignoring and Pinning Specific Packages

Some packages need to stay pinned due to compatibility constraints or pending migration work:

```json
{
  "packageRules": [
    {
      "matchPackageNames": ["webpack", "webpack-cli"],
      "enabled": false,
      "description": "Pinned at v4 until webpack 5 migration completes"
    },
    {
      "matchPackageNames": ["typescript"],
      "allowedVersions": "<5.4.0",
      "description": "Block TS 5.4 until tsconfig audit done"
    },
    {
      "matchPackagePatterns": ["^@internal/"],
      "enabled": false,
      "description": "Internal packages managed manually via changesets"
    }
  ]
}
```

You can also pin the current version of a package from the Dependency Dashboard by checking its pin checkbox — Renovate will add a `// renovate: pinned` comment to `package.json` and stop proposing updates for it.

## Debugging and Testing Renovate Config

Before deploying to a live repo, validate your configuration locally:

```bash
# Validate renovate.json schema
npm install -g renovate
renovate-config-validator renovate.json

# Dry run against a repository (no PRs created)
LOG_LEVEL=debug \
RENOVATE_TOKEN=your-github-token \
renovate --dry-run=lookup yourorg/yourrepo 2>&1 | tee renovate-dry-run.log

# Check which packages would be updated
grep "packageName\|newVersion\|currentVersion" renovate-dry-run.log | head -40
```

The dry-run output shows every package Renovate evaluated, which update it would propose, and why it was skipped (version constraint, minimum release age, disabled rule, etc.). Run this before pushing config changes to catch misconfigured package rules.

## Renovate with GitLab CI

Self-hosted Renovate on GitLab uses a pipeline schedule rather than GitHub Actions:

```yaml
# .gitlab-ci.yml
renovate:
  image: renovate/renovate:latest
  stage: maintenance
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  variables:
    RENOVATE_TOKEN: $RENOVATE_GITLAB_TOKEN
    RENOVATE_PLATFORM: gitlab
    RENOVATE_ENDPOINT: https://gitlab.example.com/api/v4
    LOG_LEVEL: info
  script:
    - renovate yourgroup/yourrepo
```

Create a GitLab CI/CD schedule that runs this pipeline daily at 5 AM. Store the Renovate token — a GitLab personal access token with `api` scope — as a masked CI/CD variable named `RENOVATE_GITLAB_TOKEN`.

## Using Presets for Cross-Repo Consistency

When you manage more than a handful of repositories, maintaining identical Renovate config in each one becomes a drift problem. Renovate presets solve this: define a shared config in one repo and extend it everywhere else.

Create a `renovate-config` repo at `yourorg/renovate-config` with a `default.json`:

```json
// default.json in yourorg/renovate-config
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "ACME organization default Renovate config",
  "extends": ["config:base"],
  "timezone": "America/New_York",
  "schedule": ["before 6am on Monday"],
  "prConcurrentLimit": 5,
  "prHourlyLimit": 2,
  "commitMessagePrefix": "chore(deps):",
  "labels": ["dependencies"],
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "matchDepTypes": ["devDependencies"],
      "automerge": true
    },
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboard": true,
      "dependencyDashboardApproval": true
    }
  ]
}
```

Each project repo reduces its `renovate.json` to a single line:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>yourorg/renovate-config"]
}
```

When you adjust the global schedule or add a new group rule, update `default.json` once and all repos pick it up on their next Renovate run. Projects needing overrides extend the base and add their own `packageRules` — Renovate merges them in order.

## Measuring Renovate's Impact

After a few weeks, verify Renovate is working as intended by querying GitHub for merged dependency PRs:

```bash
# Count merged Renovate PRs in the last 30 days
gh pr list \
  --repo yourorg/yourrepo \
  --state merged \
  --label dependencies \
  --json number,title,mergedAt \
  --jq 'length'

# List major updates that required approval
gh pr list \
  --repo yourorg/yourrepo \
  --state merged \
  --label dependencies \
  --search "major in:title" \
  --json number,title,mergedAt \
  | jq '.[] | {number, title, mergedAt}'
```

A healthy Renovate setup for a mid-sized JS project typically generates 5-15 merged PRs per week, with patch auto-merges happening continuously and minor/major updates batched to Monday morning. If you see zero PRs, check the Dependency Dashboard for token errors. If you see hundreds of open PRs, tighten your `prConcurrentLimit` and add more grouping rules.

## Related Reading

- [How to Automate Code Quality Gates for Remote Teams](/remote-work-tools/how-to-automate-code-quality-gates-remote-teams/)
- [How to Set Up Verdaccio Private npm Registry](/remote-work-tools/how-to-set-up-verdaccio-private-npm-registry/)
- [Best DevsSecOps Toolchain for Remote Teams](/remote-work-tools/best-devsecops-toolchain-for-remote-teams-integrating-securi/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)

---

## Related Articles

- [Best Tools for Remote Team Dependency Tracking](/remote-work-tools/best-tools-remote-team-dependency-tracking/)
- [Remote DevOps Team Dependency Update Workflow for](/remote-work-tools/remote-devops-team-dependency-update-workflow-for-coordinati/)
- [Best Remote Collaboration Tool for Technical Architects](/remote-work-tools/best-remote-collaboration-tool-for-technical-architects-docu/)
- [How to Write Async Status Updates That Managers Actually](/remote-work-tools/how-to-write-async-status-updates-that-managers-actually-read/)
- [How to Manage Multiple GitHub Accounts for Remote Work](/remote-work-tools/how-to-manage-multiple-github-accounts-remote-work/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
