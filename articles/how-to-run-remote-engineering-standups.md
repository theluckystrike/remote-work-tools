---
layout: default
title: "How to Run Remote Engineering Standups That Work"
description: "Design async and synchronous standup formats for remote engineering teams — tools, timing, templates, and anti-patterns for 5-20 person distributed teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-run-remote-engineering-standups/
categories: [guides]
tags: [remote-work-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Most remote engineering standups are either pointless status reports or anxiety-inducing performance theater. The goal of a standup is coordination — surfacing blockers and dependencies so the team can help. This guide covers both async and synchronous formats that achieve that goal without wasting time.

## First Decision: Async or Synchronous

**Choose async if:**
- Your team spans 4+ timezones
- Anyone has a timezone where a shared standup time is unreasonable (before 7am or after 8pm)
- Engineers need deep focus time in the morning
- You already have good visibility into what people are working on

**Choose synchronous if:**
- Everyone is within 3 timezones
- The team is new and building relationships matters
- You have a recurring blockers problem that async hasn't resolved
- Sprint velocity is low and coordination is the likely cause

Most teams over 6 people default to async. Most teams under 6 can make synchronous work.

## Async Format 1: Geekbot or Standuply

Geekbot and Standuply both integrate with Slack to send each engineer a DM at a configured time, collect responses, and post a summary to the team channel.

**Geekbot setup:**

```
Questions (3 is the sweet spot):
1. What did you work on yesterday?
2. What are you working on today?
3. Any blockers or help needed?

Optional add-on for remote teams:
4. Energy level today (1-5) — gives the team visibility on who might be struggling

Schedule: 9:30am in each engineer's local timezone
Deadline: 2 hours after send (posts summary at 11:30am local)
Post to: #team-standup
Format: Thread per person (not one long message)
```

**What makes async standup responses useful:**

```
Bad: "Working on the payment thing"
Good: "Continuing work on #456 — Stripe webhook retry logic.
       Should be done by EOD. Blocker: waiting for QA to confirm
       test environment is set up."

Bad: "Meetings"
Good: "Mostly in planning sessions. Did a quick fix for #460
       (null pointer in order service) which is now in code review."
```

Create a template engineers can paste and fill in:

```
Yesterday: [PR/issue #] - [brief description]
Today: [PR/issue #] - [brief description, % complete or ETA if known]
Blockers: [none | specific blocker + who can unblock]
```

**Respecting async responses:**

Engineers should not be expected to respond to standup messages in real time. The standup is a status snapshot, not a conversation. If someone has a blocker, the IC or team lead follows up in a thread — not in the standup post.

## Async Format 2: GitHub-Based Standup

For engineering teams that prefer keeping everything in GitHub:

```yaml
# .github/workflows/standup.yml
name: Daily Standup Summary
on:
  schedule:
    - cron: '0 9 * * 1-5'  # 9am UTC Mon-Fri

jobs:
  standup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate standup summary
        uses: actions/github-script@v7
        with:
          script: |
            const yesterday = new Date();
            yesterday.setDate(yesterday.getDate() - 1);
            const since = yesterday.toISOString();

            // Get all PRs updated since yesterday
            const prs = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'all',
              sort: 'updated',
              direction: 'desc',
              per_page: 20
            });

            const recentPRs = prs.data.filter(pr =>
              new Date(pr.updated_at) > yesterday
            );

            let summary = `## Engineering Standup — ${new Date().toDateString()}\n\n`;
            summary += `### PR Activity (last 24h)\n`;

            for (const pr of recentPRs) {
              const emoji = pr.state === 'closed' ? '✅' : '🔄';
              summary += `${emoji} #${pr.number}: ${pr.title} — @${pr.user.login}\n`;
            }

            // Post to Slack via webhook
            const fetch = require('node-fetch');
            await fetch(process.env.SLACK_WEBHOOK, {
              method: 'POST',
              headers: {'Content-Type': 'application/json'},
              body: JSON.stringify({text: summary})
            });
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_STANDUP_WEBHOOK }}
```

## Synchronous Format: The 15-Minute Rule

If you run synchronous standups, they must end in 15 minutes. No exceptions. When they run long, it's because they're solving problems in real time — which should happen in a different channel.

**Synchronous standup structure:**

```
[0:00 - 0:02] Facilitator opens
"Let's do standup. Three questions: yesterday, today, blockers.
 Keep it to 30 seconds each. Long discussions go in a follow-up."

[0:02 - 0:12] Round robin
Each person: "Yesterday I worked on X. Today I'm doing Y. [Blocker or none]."
NOT: "Well, so we were looking at this issue and the problem is that..."
If it takes more than 30 seconds, say "let's take that to a thread."

[0:12 - 0:15] Blocker matching
"Alice has a blocker on the auth module. Bob, can you help?
 Let's set up a 15-minute call after this."

[0:15] Done.
```

**Common synchronous standup failure modes:**

1. **Status report theater**: Engineers summarize work no one needs to hear
   Fix: Only share info the team needs to act on

2. **Problem-solving in standup**: Someone raises a bug and the team starts debugging
   Fix: "Let's take that to a follow-up — who else needs to be involved?"

3. **Waiting for latecomers**: Starting 3-5 minutes late becomes the norm
   Fix: Start at the scheduled time, latecomers join where you are

4. **Rotating facilitator confusion**: No one knows who runs it
   Fix: Alphabetical rotation, posted in the team channel every Monday

## Hybrid Format for Mixed Timezones

When part of the team can meet synchronously but others can't:

```
Format:
1. Async engineers post updates at their morning start time
2. Sync engineers meet briefly to cover their updates and read async ones
3. Single summary posted to #team-standup channel

The sync standup reads the async updates first:
"Alice posted: working on #456, no blockers.
 Bob posted: PR ready for review on #460, needs a reviewer.
 Now for those of us here: [sync updates]"

Action items from standup go into the Slack thread,
not a separate meeting.
```

## Measuring Standup Health

Signs your standup is working:
- Blockers get resolved within 2 hours of being raised
- Engineers don't repeat the same blocker two days in a row
- The async participation rate is >80%
- Standup is 15 minutes or less (sync) or responses come in by mid-morning (async)

Signs your standup needs changing:
- Same people dominate the sync standup
- Blockers raised but never followed up
- Async responses are consistently late or skipped
- Engineers say "I already know what everyone's working on from Slack/PRs"

If engineers say the last one, that's actually a success signal — your transparency is good enough that standup is redundant. In that case, reduce to 3x/week or switch to a "blockers only" format.

## Tool Comparison: Standup Platforms

Different tools handle async standups differently. Choose based on your team's existing communication stack:

| Tool | Slack Integration | Custom Questions | Analytics | Cost | Best For |
|------|-------------------|------------------|-----------|------|----------|
| **Geekbot** | Native | Unlimited | Basic | $3-8/user/mo | Slack-first teams |
| **Standuply** | Native | Custom fields | Reporting | $4-12/user/mo | Growing teams |
| **15Five** | Native | Extensive | Advanced | $5-15/user/mo | Engagement focus |
| **GitLab Standups** | Via webhook | None | Built-in | Included | DevOps teams |
| **Manual Slack Workflow** | Native | Via forms | None | Free | <10 person teams |

For small teams (under 10 engineers), a manual Slack Workflow Builder setup is free and sufficient. For teams 10-50, Geekbot or Standuply provide reliable scaling without overhead. For 50+ teams, 15Five offers analytics that justify per-user costs.

## Standup Response Quality Template

Engineers often struggle to write useful standup responses. Provide this template in your #team-standup pinned messages:

```markdown
### Yesterday
- [Issue #XXX] Brief description of what was completed
- [Issue #YYY] Another completed item (if multiple)

### Today
- [Issue #ZZZ] What I'm starting, brief description (% complete if resuming)
- Focus area: [Brief description of today's priority]

### Blockers
- None
OR
- [Specific blocker]: [Who can help] (e.g., "Waiting for DB access: ask @dba-oncall")

### FYI
- Anything teammates should know (code review needed, PR ready, etc.)
---
Example of good response:
Yesterday: #456 - Implemented Stripe webhook retry logic, merged and deployed
Today: #457 - Building invoice export feature (~30% done), should finish EOD
Blockers: Need clarity on invoice CSV schema from @product
FYI: PR #455 ready for review, straightforward auth fix
```

## Table of Contents

- [Async Standup Failure Modes and Fixes](#async-standup-failure-modes-and-fixes)
- [Engineering Standups at Different Team Sizes](#engineering-standups-at-different-team-sizes)
- [Daily](#daily)
- [Weekly](#weekly)
- [Creating a Standup Dashboard](#creating-a-standup-dashboard)
- [Standup Anti-Patterns in Remote Teams](#standup-anti-patterns-in-remote-teams)
- [Integration with Incident Response](#integration-with-incident-response)
- [Standup Intelligence Review](#standup-intelligence-review)
- [Related Reading](#related-reading)

Provide this in your onboarding docs so new engineers learn the format immediately.

## Async Standup Failure Modes and Fixes

| Problem | Symptom | Fix |
|---------|---------|-----|
| Late responses | <50% respond by deadline | Add 1-hour buffer before summarization; post reminder at midpoint |
| Vague answers | "Working on stuff" responses | Use stricter template; provide examples of good vs bad |
| No follow-up | Blockers raised but never resolved | Assign responsibility: "Tech lead reviews all blocker comments within 2h" |
| Survey fatigue | Declining response rate over months | Switch to 3x/week or "blockers only" format |
| Timezone misalignment | Some timezones never see their updates | Send survey at same UTC time, not local time |

## Engineering Standups at Different Team Sizes

**5-8 person teams:** Daily async via Slack works best. Synchronous standup once per week if timezone overlap allows.

```bash
# Daily async example
Schedule: 9:30 AM UTC (auto-sends DM)
Deadline: 11:00 AM UTC (posts summary)
Summary format: @username's update → [yesterday] [today] [blockers]
```

**10-20 person teams:** Async daily, weekly sync meeting for cross-team blockers.

```bash
# Weekly standup meeting (15 minutes, in-person or sync)
0-2 min: Facilitator reads async responses, highlights blockers
2-12 min: Blocker matching (pair people who can help)
12-15 min: Upcoming week coordination
```

**20-50 person teams:** Sub-team async daily, rotating representation in cross-team weeklies.

```markdown
# Three-team structure (Platform, Frontend, Backend)

## Daily
- Each sub-team: async standup in their Slack channel
- Format: Same template, local timezone

## Weekly
- Blocker-only async in #eng-blockers (posted by each tech lead)
- Cross-team sync (45 min): One rep from each sub-team
 - Reps rotate monthly
 - Agenda: blockers, metrics, upcoming priorities
```

## Creating a Standup Dashboard

For teams using GitHub, create a simple dashboard showing standup health:

```python
#!/usr/bin/env python3
import json
from datetime import datetime, timedelta
from github import Github

def standup_health(org_name, team_name):
 """
 Check standup participation rates for a team
 """
 g = Github(os.environ['GITHUB_TOKEN'])
 org = g.get_organization(org_name)
 team = org.get_team_by_slug(team_name)

 # Count standup PRs (assuming standups are created as PRs)
 repo = org.get_repo('team-standups')
 since = datetime.now() - timedelta(days=5)

 prs = repo.get_pulls(
 state='all',
 since=since
 )

 # Parse PR authors
 respondents = set()
 for pr in prs:
 if pr.title.startswith('Standup:'):
 respondents.add(pr.user.login)

 # Get team membership
 members = team.get_members()
 member_names = {m.login for m in members}

 # Calculate participation
 participation = len(respondents) / len(list(member_names)) * 100

 return {
 'team': team_name,
 'participation_rate': participation,
 'respondents': list(respondents),
 'missing': list(member_names - respondents)
 }
```

## Standup Anti-Patterns in Remote Teams

**Standup as Performance Review:** Engineers feel watched, responses become defensive and political. Fix: Explicitly state that standups track coordination, not performance.

**Status Report Theater:** Standups become where engineers recite work nobody asked about. Fix: Remind team that standup is for blockers; other updates go in PRs and tickets.

**Async Responses Ignored:** Engineers post async responses that nobody reads. Fix: Have the tech lead or manager explicitly acknowledge key responses in the summary.

**Same Blocker Every Day:** Engineer reports the same blocker 3 days in a row with no resolution. Fix: Any blocker reported twice gets escalated to tech lead immediately.

## Integration with Incident Response

Standups provide early warning of systemic issues. When analyzing incidents, check:

- Were blockers raised before the incident?
- Did the standup system alert on degradation?
- Could the incident have been caught via standup flow?

Document standup insights in postmortems:

```markdown
## Standup Intelligence Review

### Week of [date]
**Reported blockers:**
- "Database performance degraded" (Mon) — Issue #XXX
- "Response times high at peak hours" (Wed) — Same root cause

**Could we have caught this earlier?**
Yes — Tuesday's standup would have highlighted pattern

**Action:**
- Add automated alerting for response time degradation
- Require standup-level metrics dashboard
```

## Related Reading

- [Best Tools for Remote Team Standup Meetings 2026](/best-tools-for-remote-team-standup-meetings-2026/)
- [Geekbot vs Standuply Async Standup Comparison](/geekbot-vs-standuply-async-standup-comparison/)
- [Async Standup Format for a Remote Mobile Dev Team of 9](/async-standup-format-for-a-remote-mobile-dev-team-of-9/)

---

## Related Articles

- [Best Tools for Remote Team Async Standups in 2026](/remote-work-tools/best-tools-for-remote-team-async-standups-2026/)
- [How to Manage Standups for a Remote QA Team of 7](/remote-work-tools/how-to-manage-standups-for-a-remote-qa-team-of-7/)
- [Remote Team Async Standup Template Guide](/remote-work-tools/remote-team-async-standup-template-guide/)
- [Async Standup Format for a Remote Mobile Dev Team of 9](/remote-work-tools/async-standup-format-for-a-remote-mobile-dev-team-of-9/)
- [Best Remote Team Async Daily Check In Format Replacing](/remote-work-tools/best-remote-team-async-daily-check-in-format-replacing-standup-meetings/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
