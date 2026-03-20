---
layout: default
title: "Example Linear API query for OKR progress"
description: "A practical guide to implementing OKR tracking for remote and distributed engineering teams. Includes code examples, tool comparisons, and."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-okr-tracking-system-for-distributed-engineerin/
categories: [guides, workflows]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Setting up an effective OKR (Objectives and Key Results) tracking system for distributed engineering teams requires more than adopting a tool. You need clear alignment between team autonomy and organizational goals, transparent progress visibility across time zones, and automated workflows that reduce tracking overhead. This guide walks you through building an OKR tracking system that actually works for remote engineering teams in 2026.

## Why OKRs Need Different Handling for Distributed Teams

In co-located teams, you can walk over to someone's desk and ask about their key results. In distributed teams, that casual check-in disappears. Your OKR system must compensate with:

- **Asynchronous check-ins** that document progress without requiring live meetings
- **Automatic progress aggregation** from existing tools developers already use
- **Clear ownership and accountability** visible to everyone, not just managers

Without these mechanisms, distributed OKRs drift into misalignment quickly. Engineers in Tokyo, London, and San Francisco need to see how their work connects to company goals without scheduling cross-timezone syncs.

## Step 1: Define Your OKR Hierarchy

Start with a three-tier hierarchy that mirrors how your team actually makes decisions:

```markdown
Company Objective: "Ship features that increase customer retention by 15%"

  Team Objective (Platform): "Reduce incident response time from 2 hours to 15 minutes"
    - KR1: Deploy automated runbook system (Target: 80% of P1 incidents auto-resolved)
    - KR2: Implement on-call handoff improvements (Target: Zero handoff failures per quarter)
    - KR3: Create incident post-mortem automation (Target: 90% of incidents documented within 24 hours)

  Team Objective (Frontend): "Improve application performance for enterprise customers"
    - KR1: Reduce initial load time to under 2 seconds (Target: 95th percentile)
    - KR2: Implement offline-first architecture (Target: Core features work without network)
    - KR3: Reduce JavaScript bundle size by 40% (Target: <200KB initial bundle)
```

Each engineering team should own one or more team objectives that roll up to company objectives. Individual contributors typically do not need personal OKRs at the engineering level; instead, their work should map to team key results.

## Step 2: Choose Your Tracking Stack

For distributed engineering teams, integrate with tools developers already use rather than adding a standalone OKR tool. Here are three practical approaches:

### Option A: Linear + Custom Dashboard

Linear already tracks issues and projects. You can extend it with custom properties:

```yaml
# Example Linear API query for OKR progress
query {
  issues(filter: {
    state: { name: { in: ["Done", "Released"] } },
    labels: { name: { eq: "Q1-OKR-KR2" } }
  }) {
    nodes {
      title
      completedAt
      estimate
    }
  }
}
```

Build a simple dashboard that aggregates issue completion by OKR label. This keeps engineers in their existing workflow.

### Option B: Notion + Slack Integration

Notion databases work well for OKR documentation with bidirectional Slack updates:

```javascript
// Slack webhook for weekly OKR check-in
const postOKRUpdate = async (channel, progress) => {
  const blocks = [
    {
      type: "section",
      text: {
        type: "mrkdwn",
        text: `*Weekly OKR Update* - ${progress.team}`
      }
    },
    {
      type: "section",
      fields: [
        { type: "mrkdwn", text: `*Key Result*\n${progress.kr}` },
        { type: "mrkdwn", text: `*Progress*\n${progress.percent}%` }
      ]
    }
  ];
  
  await slackClient.chat.postMessage({ channel, blocks });
};
```

This approach works well for teams that prefer lightweight, text-based updates over heavy workflow automation.

### Option C: OpenSource + Custom Pipeline

For teams that want full control, build your own tracking layer:

```python
# Simple OKR progress tracker (Python/Flask example)
from flask import Flask, jsonify, request
from datetime import datetime

app = Flask(__name__)

okrs = {
    "platform-reduce-incident-time": {
        "objective": "Reduce incident response time to 15 minutes",
        "key_results": [
            {"id": "kr1", "target": 80, "current": 65, "unit": "%"},
            {"id": "kr2", "target": 0, "current": 0, "unit": "failures"},
            {"id": "kr3", "target": 90, "current": 85, "unit": "%"}
        ],
        "owner": "platform-team",
        "check_ins": []
    }
}

@app.route('/api/okrs/<okr_id>/checkin', methods=['POST'])
def checkin(okr_id):
    data = request.json
    okrs[okr_id]["check_ins"].append({
        "date": datetime.utcnow().isoformat(),
        "kr_id": data["kr_id"],
        "value": data["value"],
        "notes": data.get("notes", "")
    })
    return jsonify({"status": "success"})

@app.route('/api/okrs/<okr_id>')
def get_progress(okr_id):
    return jsonify(okrs[okr_id])
```

This gives you complete customization but requires ongoing maintenance.

## Step 3: Establish Cadence and Rituals

Your OKR system fails without consistent rituals. For distributed teams, structure your cadence around asynchronous updates:

| Cadence | Activity | Format |
|---------|----------|--------|
| Weekly | Quick progress update | 2-sentence Slack message per key result |
| Bi-weekly | OKR review meeting | 30 minutes, rotating presenter per team |
| Monthly | Alignment check | Asynchronous document review in Notion/Confluence |
| Quarterly | Retro and planning | Full team session, document learnings |

Weekly updates should take under 5 minutes per person. If they take longer, your key results are too granular or your tracking too manual.

## Step 4: Automate Progress Tracking

Manual OKR updates are the biggest failure point. Connect your tracking to existing data sources:

```yaml
# Example: GitHub Actions workflow for code contribution tracking
name: OKR Progress Sync
on:
  pull_request:
    types: [closed]

jobs:
  track-progress:
    runs-on: ubuntu-latest
    steps:
      - name: Extract OKR labels
        id: extract
        run: |
          LABELS=${{ github.event.pull_request.labels }}
          echo "okr_labels=$(echo $LABELS | grep -o 'OKR-[A-Z0-9]*' | tr '\n' ',')" >> $GITHUB_OUTPUT
      
      - name: Update OKR dashboard
        if: steps.extract.outputs.okr_labels
        run: |
          # Call your OKR API to increment progress
          curl -X POST $OKR_API/track \
            -d "pr=${{ github.event.pull_request.html_url }}" \
            -d "labels=${{ steps.extract.outputs.okr_labels }}"
```

This automation captures engineering output without requiring engineers to manually log their progress twice.

## Common Pitfalls to Avoid

**Setting too many key results.** Stick to 3-5 key results per objective. More than that dilutes focus and increases tracking overhead.

**Measuring output instead of outcomes.** "Ship 10 features" is an output. "Increase conversion by 10%" is an outcome. Key results should measure impact, not activity.

**Changing OKRs mid-quarter constantly.** Some adjustment is healthy, but if you're rewriting OKRs monthly, you lack strategic clarity. Establish 70% of your OKRs at quarter start; allow 30% flex for emerging priorities.

**Requiring daily standups about OKRs.** This defeats the purpose of async work. Use written updates that people consume on their own schedule.

## Measuring Success

Track these metrics to know if your OKR system is working:

- Update compliance rate: What percentage of key results receive weekly updates? Target: 80%+
- Goal achievement rate: What percentage of key results reach their target? Target: 60-70% (100% means you're sandbagging)
- Time spent on tracking: How many hours per week does the team spend on OKR-related activities? Target: <30 minutes total

If your teams are spending hours weekly on OKR administration, your system needs simplification rather than more features.

Start with the simplest tracking that provides adequate visibility, then add automation as you identify friction points. The best OKR system for distributed engineering teams is the one that fades into the background while keeping everyone aligned.

---


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [Remote Sales Team Commission Tracking Tool for.](/remote-work-tools/remote-sales-team-commission-tracking-tool-for-distributed-s/)
- [OKR Tracking for a Remote Product Team of 12 People](/remote-work-tools/okr-tracking-for-a-remote-product-team-of-12-people/)
- [Best Tool for Tracking Remote Team Goals and Key Results.](/remote-work-tools/best-tool-for-tracking-remote-team-goals-and-key-results-weekly/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
