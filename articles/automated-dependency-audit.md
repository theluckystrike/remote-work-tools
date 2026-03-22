---
layout: default
title: "How to Create Automated Dependency Audit"
description: "Build automated dependency audit workflows using Dependabot, Renovate, and custom scripts to keep npm, pip, Go, and Ruby packages current with CVE."
date: 2026-03-22
author: theluckystrike
permalink: /automated-dependency-audit/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Create Automated Dependency Audit

Outdated dependencies are the leading source of known vulnerabilities in application code. Most teams update dependencies reactively (when something breaks) rather than proactively. Automated auditing changes this: you get a PR for every outdated or vulnerable package, with CVE details, before it becomes your problem.

---

## Approach 1: Dependabot (GitHub Native)

Dependabot is built into GitHub and requires zero infrastructure. Enable it with a config file and it opens PRs for outdated packages automatically.

**`.github/dependabot.yml`**

```yaml
version: 2
updates:
  # npm / Node.js
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "UTC"
    open-pull-requests-limit: 10
    reviewers:
      - "yourorg/backend-team"
    labels:
      - "dependencies"
      - "automated"
    commit-message:
      prefix: "chore(deps)"
    groups:
      # Group all minor/patch updates into one PR
      minor-and-patch:
        update-types:
          - "minor"
          - "patch"

  # Python
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "tuesday"
    open-pull-requests-limit: 5
    labels:
      - "dependencies"

  # Go
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5

  # Docker base images
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "ci/cd"
```

**Auto-merge patch updates** (safe for most projects):

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Auto-merge Dependabot patch PRs
on:
  pull_request:

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.actor == 'dependabot[bot]'
    steps:
      - name: Fetch Dependabot metadata
        id: meta
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Auto-merge patch updates
        if: |
          steps.meta.outputs.update-type == 'version-update:semver-patch' &&
          steps.meta.outputs.dependency-type == 'direct:production'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Approach 2: Renovate (More Powerful)

Renovate opens PRs for every ecosystem (Docker, Helm, Terraform, Ansible, etc.) with more grouping and scheduling options than Dependabot.

**`renovate.json` at repo root:**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    "schedule:weekdays",
    ":semanticCommits",
    ":automergeMinor",
    "group:allNonMajor"
  ],
  "timezone": "UTC",
  "schedule": ["before 6am on monday"],
  "labels": ["dependencies", "automated"],
  "reviewers": ["team:backend"],
  "packageRules": [
    {
      "description": "Auto-merge patch updates with passing CI",
      "matchUpdateTypes": ["patch"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["ci/test", "ci/lint"]
    },
    {
      "description": "Group all AWS SDK updates",
      "matchPackagePrefixes": ["@aws-sdk/", "aws-sdk"],
      "groupName": "AWS SDK"
    },
    {
      "description": "Group all Terraform provider updates",
      "matchManagers": ["terraform"],
      "groupName": "Terraform providers"
    },
    {
      "description": "Pin Docker digest for production images",
      "matchManagers": ["dockerfile"],
      "matchPackageNames": ["node", "python", "golang"],
      "pinDigests": true
    },
    {
      "description": "Require manual review for major updates",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "labels": ["dependencies", "breaking-change"],
      "reviewers": ["team:leads"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security", "dependencies"],
    "schedule": ["at any time"]  // CVE fixes don't wait for weekly schedule
  }
}
```

**Self-hosted Renovate on GitHub Actions:**

```yaml
# .github/workflows/renovate.yml
name: Renovate
on:
  schedule:
    - cron: "0 6 * * 1"  # Mondays at 6am UTC
  workflow_dispatch:     # allow manual trigger

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: renovatebot/github-action@v40
        with:
          token: ${{ secrets.RENOVATE_TOKEN }}
        env:
          RENOVATE_CONFIG_FILE: renovate.json
          LOG_LEVEL: info
```

---

## Approach 3: Custom Audit Script (Any CI)

For organizations without GitHub, a custom script audits dependencies and posts results to Slack:

```bash
#!/bin/bash
# audit-deps.sh — multi-ecosystem dependency audit
set -euo pipefail

SLACK_HOOK="${SLACK_WEBHOOK_URL:-}"
CRITICAL_ONLY="${1:-false}"
REPORT=""
EXIT_CODE=0

# npm audit
if [[ -f package-lock.json ]]; then
  echo "Auditing npm..."
  NPM_RESULT=$(npm audit --json 2>/dev/null || true)
  CRITICAL_COUNT=$(echo "$NPM_RESULT" | jq '.metadata.vulnerabilities.critical // 0')
  HIGH_COUNT=$(echo "$NPM_RESULT" | jq '.metadata.vulnerabilities.high // 0')
  if [[ "$CRITICAL_COUNT" -gt 0 || "$HIGH_COUNT" -gt 0 ]]; then
    REPORT="${REPORT}\n:npm: npm: ${CRITICAL_COUNT} critical, ${HIGH_COUNT} high"
    EXIT_CODE=1
  fi
fi

# pip safety check
if [[ -f requirements.txt ]]; then
  echo "Auditing Python..."
  if command -v safety &>/dev/null; then
    PY_RESULT=$(safety check -r requirements.txt --json 2>/dev/null || true)
    VULN_COUNT=$(echo "$PY_RESULT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d.get('vulnerabilities',[])))" 2>/dev/null || echo 0)
    if [[ "$VULN_COUNT" -gt 0 ]]; then
      REPORT="${REPORT}\n:python: Python: ${VULN_COUNT} vulnerable packages"
      EXIT_CODE=1
    fi
  fi
fi

# Go vulnerability check
if [[ -f go.mod ]]; then
  echo "Auditing Go..."
  if command -v govulncheck &>/dev/null; then
    GO_RESULT=$(govulncheck -json ./... 2>/dev/null || echo '{"vulns":[]}')
    VULN_COUNT=$(echo "$GO_RESULT" | jq '[.vulns? // [] | .[]] | length')
    if [[ "$VULN_COUNT" -gt 0 ]]; then
      REPORT="${REPORT}\n:go: Go: ${VULN_COUNT} vulnerable modules"
      EXIT_CODE=1
    fi
  fi
fi

# Ruby bundler audit
if [[ -f Gemfile.lock ]]; then
  echo "Auditing Ruby..."
  if command -v bundle-audit &>/dev/null; then
    if ! bundle-audit check --update -q 2>/dev/null; then
      VULN_COUNT=$(bundle-audit check --update 2>/dev/null | grep -c "Name:" || echo 1)
      REPORT="${REPORT}\n:ruby: Ruby: ${VULN_COUNT} vulnerable gems"
      EXIT_CODE=1
    fi
  fi
fi

if [[ -n "$REPORT" ]]; then
  echo -e "VULNERABILITIES FOUND:$REPORT"
  if [[ -n "$SLACK_HOOK" ]]; then
    curl -s -X POST "$SLACK_HOOK" \
      -H "Content-Type: application/json" \
      -d "{\"text\": \":rotating_light: *Dependency Audit: vulnerabilities found*\n$(echo -e "$REPORT")\"}"
  fi
fi

exit $EXIT_CODE
```

---

## Weekly Outdated Report (Not Just CVEs)

CVEs matter most, but staying ahead of major version updates avoids compounding upgrade pain:

```bash
#!/bin/bash
# outdated-report.sh — weekly summary of outdated packages

if [[ -f package.json ]]; then
  echo "=== npm outdated ==="
  npm outdated --json | jq -r 'to_entries[] | "\(.key): \(.value.current) → \(.value.latest) (\(.value.type))"' 2>/dev/null | head -20
fi

if [[ -f requirements.txt ]]; then
  echo "=== pip outdated ==="
  pip list --outdated --format=columns 2>/dev/null | head -20
fi

if [[ -f go.mod ]]; then
  echo "=== Go outdated ==="
  go list -u -m -json all 2>/dev/null | jq -r 'select(.Update) | "\(.Path): \(.Version) → \(.Update.Version)"' | head -20
fi
```

---

## Related Reading

- [How to Create Automated Security Scan Pipelines](/remote-work-tools/automated-security-scan-pipelines/)
- [How to Automate Pull Request Labeling](/remote-work-tools/automate-pull-request-labeling/)
- [Best Tools for Remote Team Secret Sharing](/remote-work-tools/remote-team-secret-sharing-tools/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
