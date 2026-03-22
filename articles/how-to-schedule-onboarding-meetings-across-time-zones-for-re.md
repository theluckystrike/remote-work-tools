---
layout: default
title: "Find overlapping work hours across three zones"
description: "Scheduling onboarding meetings across time zones presents unique challenges for remote teams. When your new hires span San Francisco, London, and Tokyo"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-schedule-onboarding-meetings-across-time-zones-for-re/
reviewed: true
score: 9
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Scheduling onboarding meetings across time zones presents unique challenges for remote teams. When your new hires span San Francisco, London, and Tokyo, finding meeting times that work for everyone requires strategy and the right tools. This guide provides practical approaches for developers and technical users who need to coordinate onboarding sessions across global distributions.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Understand the Time Zone Problem

Remote engineering teams often span three or more time zones, making synchronous meetings difficult to schedule. A meeting time that works for your San Francisco office at 9 AM PST translates to 5 PM in London and midnight in Tokyo. For onboarding, this creates friction: new team members need face-time with mentors and teammates, but forcing everyone into inconvenient hours damages morale and engagement.

The goal is finding meeting slots that minimize inconvenience while ensuring new hires receive adequate synchronous support during their first weeks.

### Step 2: Finding Optimal Meeting Times

### Manual Calculation with WorldTimeBuddy

For teams spanning two or three time zones, start with WorldTimeBuddy or a similar time zone visualizer. The key is identifying "golden hours" where most participants fall within reasonable working hours (9 AM to 6 PM local time).

Consider a team with members in:
- Pacific Time (PST/PDT): UTC-8/UTC-7
- Central European Time (CET/CEST): UTC+1/UTC+2
- India Standard Time (IST): UTC+5:30

A 10 AM PST meeting becomes 7 PM CET and 10:30 PM IST—unworkable for the Indian team. Shifting to 7 AM PST gives 3 PM CET and 6:30 PM IST, which works for European and Indian colleagues but hurts the Americas team.

### Using Cron for Time Zone Calculations

For developers comfortable with the command line, cron expressions and timezone-aware tools help calculate overlap windows:

```bash
# Find overlapping work hours across three zones
# This example checks overlap between UTC, EST, and PST work hours
TZ=UTC date "+%H:%M %Z"   # Shows current UTC time
TZ=America/Los_Angeles date "+%H:%M %PST"  # Shows PST time
TZ=Europe/London date "+%H:%M %GMT"        # Shows London time
```

A more sophisticated approach uses Python with pytz:

```python
from datetime import datetime, timedelta
import pytz

def find_overlap_windows(hours_ranges):
    """Find hours that work across all time zones."""
    # hours_ranges is dict of {timezone: (start_hour, end_hour)}
    # Returns list of overlapping hours in UTC
    overlaps = []
    for hour in range(24):
        utc_hour = hour
        all_valid = True
        for tz_name, (start, end) in hours_ranges.items():
            tz = pytz.timezone(tz_name)
            local_hour = (utc_hour + tz.utcoffset(datetime.now()).seconds//3600) % 24
            if not (start <= local_hour < end):
                all_valid = False
                break
        if all_valid:
            overlaps.append(f"{utc_hour:02d}:00 UTC")
    return overlaps

# Example: Find overlap for PST (9-18), CET (9-18), IST (9-18)
zones = {
    'America/Los_Angeles': (9, 18),
    'Europe/Berlin': (9, 18),
    'Asia/Kolkata': (9, 18)
}
print(find_overlap_windows(zones))
```

This outputs hours where all three zones have participants within working hours.

### Step 3: Tools That Handle Time Zone Complexity

### Calendar Apps with World Clock Features

Google Calendar and Outlook both support multiple time zones display. Enable this feature when creating onboarding meeting series:

1. Open Calendar Settings
2. Enable "Show secondary time zone"
3. Add relevant zones (team locations)
4. When creating events, see times in both zones simultaneously

This prevents the common mistake of scheduling meetings during sleep hours for remote teammates.

### Scheduling Tools

**Calendly** and **Calendly** alternatives let invitees select from available slots in their local time. For onboarding, create a "New Hire Orientation" event type with:

- 30 or 60-minute slots
- Buffer time between meetings
- Automatic timezone detection

**World Time Buddy** remains useful for quick visual overlap checking, especially when coordinating one-off meetings.

### Automation with Clockwise

Clockwise analyzes calendars and suggests optimal meeting times while protecting focus time. For onboarding, you can:

1. Add new hires to a specific "Onboarding" team
2. Set meeting preferences to minimize conflicts
3. Allow Clockwise to auto-schedule mentorship check-ins

The integration with Slack provides notifications when meetings are scheduled.

### Step 4: Structuring Onboarding Meetings by Time Zone Constraints

### Rotate Meeting Times Fairly

Avoid always scheduling at convenient times for one region. Create a rotation system:

| Week | Americas Time | EMEA Time | APAC Time |
|------|---------------|-----------|-----------|
| 1 | 10 AM PST | 10 AM PST | 10 AM PST |
| 2 | 7 AM PST | 7 AM PST | 7 AM PST |

This ensures no single region consistently bears the burden of early or late meetings.

### Record Meetings for Async Review

When true overlap is impossible, recording becomes essential:

- Use Zoom, Google Meet, or Teams recording features
- Save to shared drive with clear naming: `onboarding-week1-team-intro.mp4`
- Share recording link in Slack or onboarding doc
- Create a TODO for new hires to watch within 48 hours

### Split Onboarding into Regional Groups

For larger teams, separate onboarding into regional cohorts:

- Americas cohort: 9 AM - 12 PM EST
- EMEA cohort: 9 AM - 12 PM CET
- APAC cohort: 9 AM - 12 PM IST

Then schedule cross-regional "all hands" monthly rather than weekly.

### Step 5: Practical Onboarding Meeting Schedule Example

Here's a week-one schedule for a new developer joining an US-based team with European colleagues:

**Monday**
- 10:00 AM PST: 1:1 with manager (60 min)
- 2:00 PM PST: Team standup (15 min) - recorded

**Tuesday**
- 9:00 AM PST: Development environment setup walkthrough (90 min)
- 3:00 PM PST: Mentor code review session (60 min)

**Wednesday**
- 10:00 AM PST: Product overview with product manager (60 min)
- Async: Complete PagerDuty onboarding in own time

**Thursday**
- 8:00 AM PST: Architecture overview with senior engineer (60 min)
- 2:00 PM PST: Cross-team intro with European colleagues (30 min) - early for Europe, recorded for APAC absence

**Friday**
- 10:00 AM PST: Week one retro and next week planning (45 min)
- Async: Review team documentation

Notice the variation in times—this prevents any region from consistently taking inconvenient slots.

### Step 6: Handling Emergency Onboardings

Sometimes you need to bring someone on quickly. For urgent hires:

1. Check immediate availability across all zones using WorldTimeBuddy
2. Schedule short 15-minute syncs rather than long meetings
3. Rely heavily on async communication (Slack, Loom, Notion)
4. Accept that week one may have less synchronous face-time

Document this constraint so new hires understand why initial meetings are sparse.

### Step 7: Tools That Work Well

**Calendly Global Features**:
- Invite link auto-detects visitor timezone
- Shows availability in multiple zones simultaneously
- Mobile app works offline (useful for async scheduling)
- Free version supports multiple time zones
- Pricing: Free or $12/month for advanced features

**World Time Buddy**:
- Visual representation of time overlap
- Can save team time zone configurations
- Great for finding windows manually
- Free with paid options
- Pricing: Free or $6.99/month premium

**Google Calendar Secondary Timezone**:
- Already included in your existing calendar
- Shows two time zones side-by-side
- Simple and effective for most teams
- No additional cost

For most distributed teams, Google Calendar secondary zones + Calendly handles 95% of needs. Only upgrade if you're managing 20+ onboardings simultaneously.

### Step 8: Common Scheduling Mistakes

**Mistake 1: Assuming midnight is the cutoff**
A 1 AM meeting is brutal but sometimes beats forcing someone to 3 PM the night before. Most people prefer early morning (6-8 AM) to very late night.

**Mistake 2: Always optimizing for one region**
If founder is in SF, every meeting ends up at convenient Pacific times. Rotate meeting times so burden is shared.

**Mistake 3: Scheduling too many synchronous meetings**
Ambitious managers schedule 8-10 meetings in first week. Reduce to 5-6. Let new hires breathe and actually start productive work.

**Mistake 4: Not recording for async viewing**
Treat every meeting as recorded for those who miss it. Invest 2 minutes in setup. Get Zoom transcripts automatically.

**Mistake 5: Changing meeting times at the last minute**
If you move a meeting, give 48 hours notice minimum. Changing timezone math at last second confuses people.

### Step 9: Async Onboarding Materials (Complement to Meetings)

Structure your onboarding so meetings are 40% of the experience:

**Pre-meeting (async, first 2 days)**:
- Welcome email with team intro videos
- Equipment setup guides
- Codebase overview (wiki or Notion)
- First week calendar (with timezone callouts)

**During meetings (synchronous, weeks 1-2)**:
- 1:1s with key stakeholders
- Architecture and systems deep dives
- Team culture and norms
- Code review training

**Post-meeting (async, weeks 2+)**:
- Recorded walkthrough videos of critical systems
- Take-home assignments to deepen learning
- Office hours (async Q&A document updated daily)
- Peer buddy relationship (daily async check-ins)

This balances the synchronous face-time that builds relationships with asynchronous learning that happens at each person's own pace.

### Step 10: Onboarding Timeline Template

Copy this structure for your distributed team:

**Week 1 (Max 5 meetings)**:
- Day 1: Manager 1:1 (60 min, optimize for manager timezone first day)
- Day 1: Team intro video (async, record for those with conflicts)
- Day 2: Dev environment setup (60 min, accommodate afternoon for APAC)
- Day 3: Architecture overview (60 min, rotate timezone preference)
- Day 4: Code standards walkthrough (90 min, schedule for mid-week best overlap)
- Day 5: Week retro + planning (45 min, early morning for Americas)

**Week 2** (Max 4 meetings, more async work):
- Day 1-2: Async: Complete first code review exercise
- Day 3: Peer code review session (60 min, with buddy)
- Day 4: Cross-team intro (30 min, recording mandatory)
- Day 5: Week retro + next steps

**Week 3+** (Meetings as needed, heavy on productive work):
- Continue 1:1s but reduce frequency to weekly
- Async documentation review and updates
- First PR assignment (pair with experienced mentor)

This structure ensures intensive support first week while ramping to normal pace by week 3.

### Step 11: Success Metrics for Distributed Onboarding

Track these to know if your onboarding is working:

**Time to first contribution**: From start date to first merged PR or shipped work. Target: 5-7 business days. Long timelines (2+ weeks) suggest unclear expectations or technical setup issues.

**Onboarding meeting attendance**: New hire should attend 80%+ of scheduled meetings. Missing meetings signals scheduling is unrealistic for their timezone.

**Survey satisfaction**: Ask new hire on day 7 and day 30: "Rate your onboarding experience 1-5." Scores below 3 suggest rework is needed.

**Learning velocity**: Do new hires understand core systems by end of week 2? Quiz them informally. If confused, docs or meetings need improvement.

**Retention at 90 days**: Track onboarding quality by 90-day retention. If 20%+ of new hires leave within 90 days, onboarding is likely the culprit.

Good distributed onboarding gets people productive by week 3 and confident by week 6. If it takes longer, you're burning money on extended ramp-up.

### Step 12: Scaling Onboarding for Growth

This schedule works for 1-2 new hires per month. As you scale:

**5+ new hires per month**: Create a cohort model. Onboard 2-3 people simultaneously. They become each other's support network. Saves management time, improves new hire connections.

**10+ new hires per month**: Assign onboarding buddies from previous cohorts. Rotate who does buddy duty. Spreads load across team.

**20+ new hires per month**: Hire an onboarding specialist. Full-time role coordinating onboarding, tracking metrics, improving processes.

Most distributed companies stabilize at ~2-3 new hires monthly. Your current schedule scales fine. Revisit if hiring velocity increases.
---


## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**How do I get my team to adopt a new tool?**

Start with a small pilot group of willing early adopters. Let them use it for 2-3 weeks, then gather their honest feedback. Address concerns before rolling out to the full team. Forced adoption without buy-in almost always fails.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [How to Manage Remote Team Handoffs Across Time Zones: A](/remote-work-tools/how-to-manage-remote-team-handoffs-across-time-zones/)
- [Example: Finding interview slots across time zones](/remote-work-tools/remote-team-hiring-manager-training-program-for-first-time-m/)
- [How to Schedule Meetings Across 8 Hour Timezone Difference](/remote-work-tools/how-to-schedule-meetings-across-8-hour-timezone-difference-w/)
- [Example on-call schedule that uses timezone difference](/remote-work-tools/how-to-negotiate-flexible-hours-with-us-employer-when-workin/)
- [Find the first commit by a specific author](/remote-work-tools/best-practice-for-measuring-remote-onboarding-effectiveness-with-time-to-first-commit/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
