---
layout: default
title: "How to Create a Remote Team Tech Radar"
description: "Build a living technology radar for distributed teams using Thoughtworks format with Backstage or a static generator to track adopt, trial, and hold decisions"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-remote-team-tech-radar/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A tech radar is a snapshot of your team's technology decisions: what you're adopting, actively trialing, holding off on, and avoiding. For remote teams it replaces hallway conversations about "should we try X?" with a documented, searchable record. This guide builds one using a CSV + static generator approach your whole team can contribute to via pull requests.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: The Four Quadrants and Rings

Thoughtworks format uses four quadrants and four rings:

```
Quadrants:
  Languages & Frameworks  — Python, React, FastAPI
  Platforms               — AWS, k3s, Cloudflare
  Tools                   — DBeaver, Bruno, Restic
  Techniques              — Contract testing, GitOps, ADRs

Rings:
  ADOPT    — Use in production; proven in our context
  TRIAL    — Use on low-risk projects; still evaluating
  ASSESS   — Worth exploring; research phase
  HOLD     — Pause new adoption; not recommended
```

### Step 2: Option A: Static Generator (No Infrastructure)

The `build-your-own-radar` tool from Thoughtworks reads a CSV and generates an interactive radar.

```bash
# Clone the generator
git clone https://github.com/thoughtworks/build-your-own-radar.git
cd build-your-own-radar
npm install
```

```csv
# radar.csv
name,ring,quadrant,isNew,description
React,adopt,Languages & Frameworks,FALSE,"Stable choice for web UIs. Well-understood by team."
FastAPI,adopt,Languages & Frameworks,FALSE,"Python API framework. Good async support and OpenAPI generation."
Svelte,trial,Languages & Frameworks,TRUE,"Exploring for dashboards. Bundle size benefit vs React."
HTMX,assess,Languages & Frameworks,TRUE,"Interesting for reducing JS complexity in server-rendered apps."
jQuery,hold,Languages & Frameworks,FALSE,"No new projects. Migrating existing to Alpine.js."
k3s,adopt,Platforms,FALSE,"Lightweight Kubernetes for dev clusters. In production for 6 months."
Cloudflare Workers,trial,Platforms,TRUE,"Testing for edge caching and auth middleware."
Vercel,assess,Platforms,TRUE,"Evaluating for frontend deployments. Cost TBD at scale."
DBeaver,adopt,Tools,FALSE,"Standard database GUI for the team."
Bruno,adopt,Tools,TRUE,"API testing with git-stored collections. Replacing Postman."
Restic,adopt,Tools,FALSE,"Backup tool. Used for all developer machine backups."
k9s,trial,Tools,TRUE,"Terminal Kubernetes UI. Team adoption growing."
Contract Testing (Pact),trial,Techniques,TRUE,"Testing API contracts between services."
GitOps,adopt,Techniques,FALSE,"ArgoCD for deployment. Git as source of truth."
ADRs,adopt,Techniques,FALSE,"Architectural Decision Records for major choices."
Feature Flags,assess,Techniques,TRUE,"Evaluating LaunchDarkly vs self-hosted."
```

```bash
# Build and serve
npm run build
npx serve dist/

# Or host on GitHub Pages
npm run build
# Copy dist/ to your GitHub Pages repo
```

### Step 3: Option B: Backstage Tech Radar Plugin

If you're already running Backstage:

```bash
# Install the plugin
cd packages/app
yarn add @backstage-community/plugin-tech-radar

# In packages/app/src/App.tsx
import { TechRadarPage } from '@backstage-community/plugin-tech-radar';

// Add route
<Route path="/tech-radar" element={<TechRadarPage />} />
```

```typescript
// src/lib/techRadarLoader.ts
import { TechRadarLoaderResponse } from '@backstage-community/plugin-tech-radar';

export const techRadarLoader = async (): Promise<TechRadarLoaderResponse> => {
  const response = await fetch('/tech-radar.json');
  return response.json();
};
```

```json
// public/tech-radar.json
{
  "entries": [
    {
      "key": "react",
      "id": "react",
      "title": "React",
      "quadrant": "languages-frameworks",
      "description": "Stable choice for web UIs. Well-understood by team.",
      "timeline": [
        {
          "moved": 0,
          "ringId": "adopt",
          "date": "2024-01-01",
          "description": "Moved to adopt after 2 years of production use."
        }
      ]
    },
    {
      "key": "fastapi",
      "id": "fastapi",
      "title": "FastAPI",
      "quadrant": "languages-frameworks",
      "description": "Python API framework with OpenAPI generation.",
      "timeline": [
        {
          "moved": 1,
          "ringId": "adopt",
          "date": "2025-06-01",
          "description": "Moved from trial to adopt."
        },
        {
          "moved": 0,
          "ringId": "trial",
          "date": "2024-09-01",
          "description": "Started evaluation."
        }
      ]
    }
  ],
  "quadrants": [
    {"id": "languages-frameworks", "name": "Languages & Frameworks"},
    {"id": "platforms", "name": "Platforms"},
    {"id": "tools", "name": "Tools"},
    {"id": "techniques", "name": "Techniques"}
  ],
  "rings": [
    {"id": "adopt", "name": "ADOPT", "color": "#5BA300"},
    {"id": "trial", "name": "TRIAL", "color": "#009EB0"},
    {"id": "assess", "name": "ASSESS", "color": "#C7BA00"},
    {"id": "hold", "name": "HOLD", "color": "#E09B96"}
  ]
}
```

### Step 4: Contributing Process for Remote Teams

The radar is most valuable when the whole team contributes. Use a GitHub PR workflow:

```bash
# Team member wants to add a new entry
git checkout -b radar/add-bruno-api-testing
# Edit radar.csv or tech-radar.json
git commit -m "radar: add Bruno to TRIAL (tools)"
# Open PR with description explaining the recommendation
```

PR template for radar changes:

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE/radar_entry.md -->
### Step 5: Tech Radar Entry

**Technology:** [name]
**Proposed ring:** ADOPT / TRIAL / ASSESS / HOLD
**Quadrant:** Languages & Frameworks / Platforms / Tools / Techniques

### Context
Why is this relevant to our team right now?

### Experience
Have we used this? In what project/context?

### Recommendation
Why this ring placement?

### Risks / Concerns
What should we watch out for?
```

### Step 6: Automated Publishing

```yaml
# .github/workflows/radar.yml
name: Publish Tech Radar

on:
  push:
    branches: [main]
    paths:
      - 'radar.csv'
      - 'tech-radar.json'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Build radar
        run: |
          npx --yes build-your-own-radar
          cp radar.csv dist/

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./dist
          publish_branch: gh-pages
```

### Step 7: Radar Review Cadence

Schedule a quarterly async radar review:

```markdown
# Tech Radar Review — Q2 2026

**Format:** Async-first (GitHub PR comments), 30-min sync to resolve disagreements

**Timeline:**
- Week 1: Submit PRs for new entries or ring changes
- Week 2: Review and comment on PRs
- Week 3: Merge consensus PRs; flag disagreements
- Week 4: Sync call (optional, only if needed)

**Questions to answer per entry:**
1. Has our experience changed since last quarter?
2. Are there new risks or alternatives?
3. Does the ring still reflect our actual usage?
```

### Step 8: Ring Change Log

Track changes in `CHANGELOG.md` alongside the radar:

```markdown
# Radar Changelog

### Step 9: 2026-Q2

### Moved to ADOPT
- Bruno (Tools) — API testing with git-stored collections.
  Team-wide adoption complete. Postman decommissioned.
- Contract Testing/Pact (Techniques) — Used in 3 services.

### Moved to TRIAL
- Svelte (Languages) — Two dashboards in production.
  Evaluating bundle size gains.

### Moved to HOLD
- AWS Lambda@Edge — Complexity too high for our team size.
  Replaced with Cloudflare Workers for edge logic.

### New ASSESS entries
- Bun (Languages) — Node.js alternative, watching for ecosystem maturity.
```

### Step 10: Deciding What Goes on the Radar

Not everything belongs on a tech radar. A common mistake is listing every library, every SaaS tool, and every language variant — the radar becomes noise and engineers stop consulting it. Apply a filter:

**Include if:**
- The team has made or is considering a deliberate adoption decision
- There is genuine disagreement or uncertainty about whether to use it
- The tool/technique has cross-team or multi-project implications
- An entry would save the next person from relitigating a debate you already had

**Exclude if:**
- It is a minor dependency with no real decision behind it (e.g., `lodash`)
- Only one engineer on one project uses it and there is no wider intent
- The decision is already universally settled (e.g., Git for version control)

When in doubt, write it as an ADR first. If the ADR matters enough to reference repeatedly, promote it to the radar.

### Step 11: Linking the Radar to ADRs

Tech radar entries gain credibility when backed by an Architectural Decision Record. Add an `adr` field to your CSV:

```csv
name,ring,quadrant,isNew,description,adr
Contract Testing (Pact),trial,Techniques,TRUE,"Testing API contracts between services. Reduces integration test flakiness.",ADR-014
GitOps,adopt,Techniques,FALSE,"ArgoCD for deployment. Git as source of truth.",ADR-009
AWS Lambda@Edge,hold,Platforms,FALSE,"Complexity too high for our team size. See ADR for alternatives.",ADR-022
```

When the Backstage plugin renders entries, the `description` field can include a markdown link:

```json
{
  "key": "gitops",
  "title": "GitOps",
  "description": "ArgoCD for all production deployments. Git as single source of truth for cluster state. [ADR-009](/docs/adr/009-gitops-with-argocd.md)",
  "timeline": [...]
}
```

This creates a traceable audit trail: you can always read the original reasoning behind a ring placement, not just the current recommendation.

### Step 12: Run Your First Radar Session

The first time a team builds a radar, the session often stalls because nobody is sure what ring to assign to a technology they have mixed feelings about. Use this facilitation format for remote teams:

**Async phase (Week 1 — 45 minutes per person):**
1. Each engineer independently lists 5–10 technologies they have a strong opinion about
2. They assign a ring (ADOPT/TRIAL/ASSESS/HOLD) and write 2–3 sentences of reasoning
3. Submit as draft PRs or a shared spreadsheet

**Synthesis phase (async, Day 1 of Week 2):**
The radar owner (typically a staff engineer or tech lead) merges duplicates, identifies disagreements, and flags entries where reviewers chose different rings for the same technology.

**Discussion phase (30-min Zoom, Week 2):**
Skip consensus items. Only discuss the flagged disagreements. Use a simple rule: if two or more engineers who have actually used the technology prefer different rings, the lower ring wins. You move to ADOPT only when the team has enough shared production experience to say "this works for us."

**Publish and celebrate (Week 2):**
Merge the PR. Post the radar link in `#engineering`. Make the first publication a moment — it signals that the team takes technology decisions seriously enough to write them down.

### Step 13: Measuring Radar Effectiveness

After two quarters, ask these questions to evaluate whether the radar is working:

- Are engineers referencing the radar in PR reviews or architecture discussions?
- Have any TRIAL entries graduated to ADOPT based on real experience?
- Are HOLD entries being respected, or are engineers still reaching for them?
- Has the radar prevented any "should we use X?" debates that would otherwise have taken a week of Slack messages?

A radar that gets consulted saves time. A radar that gets ignored is a documentation artifact. If nobody uses it, the problem is usually one of: it is not visible enough (add a link to your eng handbook front page), it is not maintained (stale entries), or it does not cover decisions the team actually faces (wrong quadrant choices for your stack).

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [ADR Tools for Remote Engineering Teams](/remote-work-tools/adr-tools-for-remote-engineering-teams/)
- [Async Decision Making with RFC Documents](/remote-work-tools/async-decision-making-with-rfc-documents-for-engineering-tea/)
- [Best Practice for Remote Team Decision Making Framework](/remote-work-tools/best-practice-for-remote-team-decision-making-framework-that/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
