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

## Key Takeaways

- **Topics covered**: the four quadrants and rings, option a: static generator (no infrastructure), option b: backstage tech radar plugin
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements
- **Trade-off analysis**: Strengths and limitations of each option discussed

## The Four Quadrants and Rings

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

## Option A: Static Generator (No Infrastructure)

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

## Option B: Backstage Tech Radar Plugin

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

## Contributing Process for Remote Teams

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
## Tech Radar Entry

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

## Automated Publishing

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

## Radar Review Cadence

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

## Ring Change Log

Track changes in `CHANGELOG.md` alongside the radar:

```markdown
# Radar Changelog

## 2026-Q2

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

## Related Reading

- [ADR Tools for Remote Engineering Teams](/remote-work-tools/adr-tools-for-remote-engineering-teams/)
- [Async Decision Making with RFC Documents](/remote-work-tools/async-decision-making-with-rfc-documents-for-engineering-tea/)
- [Best Practice for Remote Team Decision Making Framework](/remote-work-tools/best-practice-for-remote-team-decision-making-framework-that/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
