---
layout: default
title: "How to Automate Pull Request Labeling"
description: "Auto-label GitHub pull requests by changed files, PR size, branch name, and title patterns using Actions workflows, Labeler, and custom scripts for remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /automate-pull-request-labeling/
categories: [guides]
reviewed: true
score: 7
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Automate Pull Request Labeling

Labels tell your team what a pull request is before they open it. Without automation, labels get applied inconsistently or not at all. With a few workflow files, every PR gets labeled the moment it's opened by changed files, size, and title convention.

---

## Approach 1: GitHub's Official Labeler Action

**`.github/labeler.yml`**

```yaml
frontend:
  - changed-files:
    - any-glob-to-any-file:
      - "src/frontend/**"
      - "app/assets/**"
      - "*.css"

backend:
  - changed-files:
    - any-glob-to-any-file:
      - "src/api/**"
      - "app/controllers/**"
      - "app/models/**"

infrastructure:
  - changed-files:
    - any-glob-to-any-file:
      - "terraform/**"
      - "kubernetes/**"
      - "docker-compose*.yml"
      - "Dockerfile*"

documentation:
  - changed-files:
    - any-glob-to-any-file:
      - "docs/**"
      - "**/*.md"

tests:
  - changed-files:
    - any-glob-to-any-file:
      - "**/*.test.*"
      - "**/*.spec.*"
      - "tests/**"

dependencies:
  - changed-files:
    - any-glob-to-any-file:
      - "package.json"
      - "package-lock.json"
      - "yarn.lock"
      - "go.mod"
      - "requirements*.txt"
```

**`.github/workflows/labeler.yml`**

```yaml
name: Label Pull Request
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
          configuration-path: .github/labeler.yml
          sync-labels: true
```

---

## Approach 2: Label by PR Size

```yaml
name: PR Size Label
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  pull-requests: write

jobs:
  size:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const pr = context.payload.pull_request;
            const total = pr.additions + pr.deletions;

            const sizeLabels = {
              "size/XS": total <= 10,
              "size/S":  total > 10  && total <= 100,
              "size/M":  total > 100 && total <= 500,
              "size/L":  total > 500 && total <= 1000,
              "size/XL": total > 1000,
            };

            const allSizeLabels = Object.keys(sizeLabels);
            const newLabel = Object.entries(sizeLabels)
              .find(([, matches]) => matches)?.[0];

            const { data: currentLabels } = await github.rest.issues.listLabelsOnIssue({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: pr.number,
            });

            for (const label of currentLabels.map(l => l.name).filter(l => allSizeLabels.includes(l))) {
              await github.rest.issues.removeLabel({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: pr.number,
                name: label,
              });
            }

            try {
              await github.rest.issues.createLabel({
                owner: context.repo.owner,
                repo: context.repo.repo,
                name: newLabel,
                color: total <= 100 ? "0e8a16" : total <= 500 ? "e4e669" : "b60205",
              });
            } catch (e) { /* Label already exists */ }

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: pr.number,
              labels: [newLabel],
            });
```

---

## Approach 3: Label by Conventional Commit Title

```yaml
name: Conventional PR Labels
on:
  pull_request:
    types: [opened, edited, synchronize]

permissions:
  pull-requests: write

jobs:
  label:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const pr = context.payload.pull_request;
            const title = pr.title.toLowerCase();

            const typeMap = {
              "feat":     { label: "feature",      color: "0075ca" },
              "fix":      { label: "bug",           color: "d73a4a" },
              "chore":    { label: "chore",         color: "e4e669" },
              "docs":     { label: "documentation", color: "0075ca" },
              "refactor": { label: "refactor",      color: "7057ff" },
              "perf":     { label: "performance",   color: "e4e669" },
              "test":     { label: "tests",         color: "0e8a16" },
              "ci":       { label: "ci/cd",         color: "0052cc" },
            };

            const labelsToAdd = [];
            for (const [prefix, { label, color }] of Object.entries(typeMap)) {
              if (title.startsWith(prefix + ":") || title.startsWith(prefix + "(")) {
                labelsToAdd.push({ name: label, color });
                break;
              }
            }

            if (title.includes("!:") || title.includes("breaking")) {
              labelsToAdd.push({ name: "breaking-change", color: "b60205" });
            }

            for (const { name, color } of labelsToAdd) {
              try {
                await github.rest.issues.createLabel({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  name, color,
                });
              } catch (e) { /* exists */ }

              await github.rest.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: pr.number,
                labels: [name],
              });
            }
```

---

## Bootstrap Labels in a New Repo

```bash
#!/bin/bash
REPO="${GITHUB_REPOSITORY:-yourorg/yourrepo}"

create_label() {
  gh label create "$1" --color "$2" --description "$3" --repo "$REPO" --force 2>/dev/null || true
}

create_label "feature"        "0075ca" "New feature"
create_label "bug"            "d73a4a" "Bug fix"
create_label "chore"          "e4e669" "Maintenance task"
create_label "documentation"  "0075ca" "Documentation update"
create_label "refactor"       "7057ff" "Code refactoring"
create_label "tests"          "0e8a16" "Test additions or fixes"
create_label "ci/cd"          "0052cc" "CI/CD pipeline changes"
create_label "breaking-change" "b60205" "Breaking API or behavior change"
create_label "size/XS"        "0e8a16" "< 10 lines"
create_label "size/S"         "0e8a16" "10-100 lines"
create_label "size/M"         "e4e669" "100-500 lines"
create_label "size/L"         "d93f0b" "500-1000 lines"
create_label "size/XL"        "b60205" "> 1000 lines"
create_label "dependencies"   "0075ca" "Dependency updates"

echo "Labels created in $REPO"
```

---

## Using Labels in Changelog Generation

```yaml
# .github/release-please.yml
changelog-sections:
  - type: feature
    section: "Features"
  - type: bug
    section: "Bug Fixes"
  - type: breaking-change
    section: "Breaking Changes"
  - type: performance
    section: "Performance"
  - type: dependencies
    section: "Dependencies"
```

---

## Related Reading

- [How to Create Automated Security Scan Pipelines](/remote-work-tools/automated-security-scan-pipelines/)
- [Best Tools for Remote Team Changelog Review](/remote-work-tools/remote-team-changelog-review-tools/)
- [How to Create Automated Dependency Audit](/remote-work-tools/automated-dependency-audit/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
