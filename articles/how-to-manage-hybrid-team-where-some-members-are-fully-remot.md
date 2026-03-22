---
layout: default
title: "Generate weekly team activity report from GitHub"
description: "A practical guide for developers and power users on managing hybrid teams with permanent remote members. Includes automation scripts, workflow"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /how-to-manage-hybrid-team-where-some-members-are-fully-remot/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

Hybrid teams with split remote and in-office members need explicit communication norms, video-first meetings, and documentation-first workflows to prevent information silos. Establish core hours for synchronous overlap, implement async-first standups using GitHub Actions or Slack, and make all meeting rooms video-conference ready with equal participation cues. This guide provides practical systems and code examples for managing teams that work across locations without enterprise tools.

## The Asynchronous-First Foundation

When your team splits between permanent remote and office locations, synchronous collaboration becomes expensive. Office-based team members can naturally collaborate in real-time, but remote members face timezone constraints, technology friction, and isolation. The solution is building asynchronous workflows that treat all locations equally.

### Define Explicit Response Time Expectations

Create clear guidelines for communication responsiveness:

```typescript
// Communication response expectations by channel
interface TeamCommunicationNorm {
  channel: 'slack' | 'email' | 'github' | 'emergency';
  responseTimeHours: number;
  examples: string[];
}

const communicationNorms: TeamCommunicationNorm[] = [
  {
    channel: 'slack',
    responseTimeHours: 4,
    examples: ['quick questions', 'status updates', 'non-blocking issues']
  },
  {
    channel: 'email',
    responseTimeHours: 24,
    examples: ['formal requests', 'documentation', 'external communication']
  },
  {
    channel: 'github',
    responseTimeHours: 8,
    examples: ['code reviews', 'PR feedback', 'issue triage']
  },
  {
    channel: 'emergency',
    responseTimeHours: 0.5,
    examples: ['production outages', 'security incidents', 'critical bugs']
  }
];
```

This explicitly defines expectations so remote team members aren't expected to respond instantly while office workers don't feel ignored.

## Build Transparent Work Visibility

Remote team members suffer from reduced visibility into what others are working on. Rather than relying on status updates or frequent check-ins, implement systems that make work visible through existing tools.

### Automate Status Updates from Code Activity

Pull activity data directly from your development tools:

```python
# Generate weekly team activity report from GitHub
import requests
from datetime import datetime, timedelta

def generate_team_activity_report(github_token, org, team_slug):
    headers = {"Authorization": f"token {github_token}"}
    base_url = "https://api.github.com"

    # Get team members
    members_url = f"{base_url}/orgs/{org}/teams/{team_slug}/members"
    members = requests.get(members_url, headers=headers).json()

    report = []
    week_ago = (datetime.now() - timedelta(days=7)).isoformat()

    for member in members:
        username = member["login"]

        # Get PRs created
        prs_url = f"{base_url}/search/issues?q=author:{username}+is:pr+created:>{week_ago}"
        prs = requests.get(prs_url, headers=headers).json()

        # Get reviews done
        reviews_url = f"{base_url}/search/issues?q=reviewer:{username}+is:pr+updated:>{week_ago}"
        reviews = requests.get(reviews_url, headers=headers).json()

        report.append({
            "username": username,
            "prs_created": prs.get("total_count", 0),
            "reviews_done": reviews.get("total_count", 0)
        })

    return report
```

This script generates a factual, objective view of contribution without requiring anyone to manually report their work.

## Create Location-Agnostic Meeting Rules

Meetings are where hybrid teams most frequently fail remote participants. Implement these rules systematically:

### The No-Office-Only Rule

Any meeting that involves decision-making or problem-solving must include remote participants by default. This sounds obvious, but teams frequently default to in-person discussions.

```bash
# Create a Slack reminder for meeting inclusivity
# Add to your team workflow or GitHub Actions

name: Meeting Accessibility Check
on:
  schedule:
    - cron: '0 9 * * 1'  # Monday morning
jobs:
  check-meetings:
    runs-on: ubuntu-latest
    steps:
      - name: Check upcoming meetings
        run: |
          echo "## Meeting Inclusivity Checklist"
          echo "- [ ] Are remote participants added to all calendar invites?"
          echo "- [ ] Is there a Zoom/Meet link for every meeting?"
          echo "- [ ] Will someone be designated to represent remote perspectives?"
          echo "- [ ] Will meeting notes be shared within 24 hours?"
```

### Rotate Host Responsibilities

Rotate meeting help between remote and office team members to ensure both perspectives get equal airtime.

## Implement Pair Programming Across Locations

Remote developers miss the spontaneous pair programming that happens in offices. Actively create opportunities:

```javascript
// Schedule async pair programming sessions
const pairSessionConfig = {
  durationMinutes: 90,
  frequency: 'weekly',
  rotation: 'random', // or 'round-robin'
  tools: ['LiveShare', 'Tuple', 'Screen sharing'],

  // Time slots that work across time zones
  preferredSlots: [
    { utcStart: 14, utcEnd: 15.5 },  // 9am PST / 6pm CET
    { utcStart: 20, utcEnd: 21.5 },  // 12pm PST / 9pm CET
  ]
};

function findOptimalPairingSlots(members) {
  // Find overlapping hours across all member timezones
  const memberTimezones = members.map(m => m.timezone);
  // Return slots where everyone has at least 2 hours of overlap
  return pairSessionConfig.preferredSlots.filter(slot =>
    memberTimezones.every(tz => isWithinWorkingHours(slot, tz))
  );
}
```

## Handle Knowledge Transfer Proactively

Office workers absorb knowledge through overheard conversations and informal chats. Remote workers miss this entirely. Close the gap through deliberate documentation:

### The Decision Log Practice

Every significant decision should be documented before or immediately after it's made:

```markdown
# Decision Record Template

## [Title]
**Date:** YYYY-MM-DD
**Deciders:** @person1, @person2
**Status:** [Proposed | Decided | Deprecated]

### Context
What problem are we solving?

### Options Considered
1. Option A - pros/cons
2. Option B - pros/cons

### Decision
What did we choose and why?

### Consequences
What happens next?
```

Use GitHub Discussions, Notion, or a dedicated channel to maintain this log. Make it searchable so remote team members can find past decisions without asking.

## Establish Clear Documentation Locations

Create a single source of truth for team knowledge:

| Document Type | Location | Why |
|---------------|----------|-----|
| API Docs | GitHub README / Swagger | Version controlled, searchable |
| Team Processes | Notion / Confluence | Living documents |
| Project Status | Linear / GitHub Projects | Real-time visibility |
| Meeting Notes | Shared Drive / GitHub Wiki | Searchable, persistent |
| Onboarding | Notion / GitBook | New member accessibility |

Resist the temptation to spread knowledge across multiple tools. The more places you have, the harder it is for remote members to find information.

## Practical Remote Team Health Checks

Monitor team health through automated surveys that don't create busywork:

```javascript
// Weekly pulse check automation
const pulseQuestions = [
  {
    id: 'communication',
    text: 'Did you get the information you needed this week?',
    scale: 1-5
  },
  {
    id: 'collaboration',
    text: 'Were you able to collaborate effectively with your team?',
    scale: 1-5
  },
  {
    id: 'blockers',
    text: 'Do you have any blockers preventing you from doing your best work?',
    scale: 'free-text'
  }
];

// Aggregate results by location to identify disparities
function analyzeByLocation(responses) {
  return {
    remote: responses.filter(r => r.location === 'remote'),
    office: responses.filter(r => r.location === 'office')
  };
}
```

Compare scores between remote and office team members quarterly. If remote members consistently score lower on collaboration questions, investigate why.

## Onboarding Permanent Remote Members

Onboarding remote employees requires extra structure:

1. Week 1: Set up all accounts, complete security training, and run through codebase architecture
2. Week 2: Pair program with a buddy on small tasks, attend all team meetings
3. Week 3-4: Take on meaningful work with code review from multiple team members
4. Monthly: Check-in with manager on integration, tools, and process effectiveness

Document the entire onboarding process so remote hires can reference it later.



## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Does GitHub offer a free tier?**

Most major tools offer some form of free tier or trial period. Check GitHub's current pricing page for the latest free tier details, as these change frequently. Free tiers typically have usage limits that work for evaluation but may not be sufficient for daily professional use.


**How do I get my team to adopt a new tool?**

Start with a small pilot group of willing early adopters. Let them use it for 2-3 weeks, then gather their honest feedback. Address concerns before rolling out to the full team. Forced adoption without buy-in almost always fails.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


## Related Articles

- [Example: Minimum device requirements for team members](/remote-work-tools/how-to-implement-device-management-policy-for-fully-remote-s/)
- [How to Transition Team Rituals from Fully Remote to Hybrid](/remote-work-tools/how-to-transition-team-rituals-from-fully-remote-to-hybrid-f/)
- [.github/ISSUE_TEMPLATE/onboarding.yml](/remote-work-tools/hybrid-team-onboarding-process-template-for-new-hires-splitting-time-office-and-home/)
- [How to Manage Multiple GitHub Accounts for Remote Work](/remote-work-tools/how-to-manage-multiple-github-accounts-remote-work/)
- [How to Use AI Tools to Generate Remote Team Meeting.](/remote-work-tools/how-to-use-ai-tool-to-generate-remote-team-meeting-agendas-f/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
