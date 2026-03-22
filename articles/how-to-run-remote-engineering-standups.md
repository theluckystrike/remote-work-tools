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

## Related Reading

- [Best Tools for Remote Team Standup Meetings 2026](/best-tools-for-remote-team-standup-meetings-2026/)
- [Geekbot vs Standuply Async Standup Comparison](/geekbot-vs-standuply-async-standup-comparison/)
- [Async Standup Format for a Remote Mobile Dev Team of 9](/async-standup-format-for-a-remote-mobile-dev-team-of-9/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
