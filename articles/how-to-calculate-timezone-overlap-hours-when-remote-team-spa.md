---
layout: default
title: "Team hours (as datetime.time objects converted to hours)"
description: "A practical guide for developers and power users to calculate timezone overlap hours between Asia and Americas using code and proven formulas"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-calculate-timezone-overlap-hours-when-remote-team-spa/
reviewed: true
score: 8
voice-checked: true
categories: [guides]
intent-checked: true
tags: [remote-work-tools, remote-work]
---

Asia-Americas distributed teams typically find only 2-4 hours of real-time overlap (usually early morning Americas time, late evening Asia time), making asynchronous communication the default and scheduling full-team standups nearly impossible without significant timezone sacrifice. Python or JavaScript timezone libraries can calculate exact overlap windows factoring in daylight saving time shifts, daylight hours per location, and workday availability to identify the optimal narrow window for synchronous work. Understanding that most teams operate on async-first with occasional 1:1 handoff meetings during overlap hours allows better planning than forcing synchronous collaboration that requires one region's team to work outside standard hours.

## Understanding the Time Zone Gap

The time difference between major Asian and American cities ranges from 12 to 20 hours, depending on specific locations and daylight saving time adjustments. This gap means that when one region is at the start of its workday, the other is either ending theirs or is already in the evening hours.

For example, consider a team distributed across Tokyo (JST, UTC+9) and San Francisco (PST, UTC-8):

- Tokyo: 9:00 AM Monday
- San Francisco: 4:00 PM Sunday

The reverse scenario shows similar challenges. When San Francisco begins its day at 9:00 AM PST, Tokyo is already at 1:00 AM the following day.

## The Overlap Formula

At its core, calculating timezone overlap requires understanding each team's working hours and finding the intersection. Here's the fundamental formula:

```
Overlap Start = max(TeamA_Start, TeamB_Start - TimeDifference)
Overlap End = min(TeamA_End, TeamB_End - TimeDifference)
Overlap Hours = Overlap End - Overlap Start
```

A practical JavaScript implementation helps visualize this:

```javascript
function calculateOverlap(teamA, teamB) {
  // team structure: { startHour: number, endHour: number, offset: number }
  // offset is UTC offset in hours

  const getLocalHour = (hour, offset, referenceOffset) => {
    let localHour = hour - (offset - referenceOffset);
    if (localHour < 0) localHour += 24;
    if (localHour >= 24) localHour -= 24;
    return localHour;
  };

  const aStartLocal = getLocalHour(teamA.startHour, teamA.offset, 0);
  const aEndLocal = getLocalHour(teamA.endHour, teamA.offset, 0);
  const bStartLocal = getLocalHour(teamB.startHour, teamB.offset, 0);
  const bEndLocal = getLocalHour(teamB.endHour, teamB.offset, 0);

  const overlapStart = Math.max(aStartLocal, bStartLocal);
  const overlapEnd = Math.min(aEndLocal, bEndLocal);

  return {
    start: overlapStart,
    end: overlapEnd,
    hours: Math.max(0, overlapEnd - overlapStart)
  };
}

// Example: Tokyo (9-18, +9) vs San Francisco (9-18, -8)
const tokyo = { startHour: 9, endHour: 18, offset: 9 };
const sanFrancisco = { startHour: 9, endHour: 18, offset: -8 };

console.log(calculateOverlap(tokyo, sanFrancisco));
// Output shows the overlap window
```

## Practical Example: Asia-Americas Team Scheduling

Let's walk through a concrete scenario involving three team locations: Singapore, Bangalore, and Austin.

### Team Configuration

- Singapore (SGT): UTC+8, working hours 9:00 AM - 6:00 PM
- Bangalore (IST): UTC+5:30, working hours 9:30 AM - 6:30 PM
- Austin (CST): UTC-6, working hours 8:00 AM - 5:00 PM

### Calculating Pairwise Overlaps

Using Python, we can determine all overlap windows:

```python
from datetime import datetime, timedelta

def calculate_overlap(start_a, end_a, offset_a, start_b, end_b, offset_b):
    """Calculate overlap in hours between two timezones."""
    # Convert to UTC
    utc_start_a = start_a - timedelta(hours=offset_a)
    utc_end_a = end_a - timedelta(hours=offset_a)
    utc_start_b = start_b - timedelta(hours=offset_b)
    utc_end_b = end_b - timedelta(hours=offset_b)

    # Find UTC overlap
    utc_overlap_start = max(utc_start_a, utc_start_b)
    utc_overlap_end = min(utc_end_a, utc_end_b)

    overlap_hours = (utc_overlap_end - utc_overlap_start).total_seconds() / 3600
    return max(0, overlap_hours), utc_overlap_start, utc_overlap_end

# Team hours (as datetime.time objects converted to hours)
singapore = (9, 18, 8)      # 9 AM - 6 PM, UTC+8
bangalore = (9.5, 18.5, 5.5) # 9:30 AM - 6:30 PM, UTC+5:30
austin = (8, 17, -6)        # 8 AM - 5 PM, UTC-6

# Calculate Singapore-Austin overlap
overlap_sg_aus, start_utc, end_utc = calculate_overlap(
    timedelta(hours=singapore[0]), timedelta(hours=singapore[1]), singapore[2],
    timedelta(hours=austin[0]), timedelta(hours=austin[1]), austin[2]
)

print(f"Singapore-Austin overlap: {overlap_sg_aus} hours")
print(f"UTC window: {start_utc} to {end_utc}")
```

This calculation reveals that Singapore and Austin have approximately 2-3 hours of overlap, typically occurring when Austin begins its workday and Singapore approaches its evening hours.

## Strategies for Maximizing Collaboration

Once you understand your overlap windows, several strategies help maximize team productivity:

Rotate Meeting Times: Distribute the inconvenience by rotating meeting times across time zones. If your overlap window falls at 4:00 PM in one location, schedule some meetings at that time and others at a more reasonable hour for other team members.

Asynchronous-First Communication: Reduce reliance on synchronous meetings by documenting decisions thoroughly. Use collaborative tools that support async workflows, allowing team members to contribute on their own schedules.

Core Collaboration Windows: Designate a smaller "core hours" window where everyone should be available, typically 1-2 hours, and protect this time for high-bandwidth collaboration like code reviews or planning sessions.

Flexible Working Hours: Allow team members to adjust their schedules within reasonable bounds. Someone in Tokyo might start at 10:00 AM instead of 9:00 AM to align better with the Americas team.

## Using Timezone Libraries

For production applications, use established libraries rather than implementing your own calculations. The `moment-timezone` and `date-fns-tz` libraries handle edge cases including daylight saving time transitions:

```javascript
const { format, tz } = require('date-fns-tz');

function findBestMeetingSlot(locations, durationHours = 1) {
  const slots = [];

  // Check each hour over a 24-hour period
  for (let hour = 0; hour < 24; hour++) {
    let allInWorkHours = true;

    for (const loc of locations) {
      const localTime = tz(new Date().setHours(hour, 0, 0, 0), loc.timezone);
      const localHour = localTime.getHours();

      if (localHour < loc.workStart || localHour >= loc.workEnd) {
        allInWorkHours = false;
        break;
      }
    }

    if (allInWorkHours) {
      slots.push(hour);
    }
  }

  return slots;
}
```

## Common Pitfalls to Avoid

When calculating timezone overlaps, watch for these frequent mistakes:

- Ignoring Daylight Saving Time: Always use IANA timezone identifiers (like "Asia/Tokyo" or "America/Los_Angeles") rather than fixed UTC offsets, as DST changes affect offsets throughout the year.
- Assuming Same Working Hours: Not all teams work 9-to-5. Confirm actual working hours with team members, as flexibility varies by culture and role.
- Forgetting Weekends: Some team members might work weekends occasionally. Factor in weekend preferences when scheduling recurring meetings.

## Real-World Examples: Common Asia-Americas Configurations

**Tokyo + San Francisco (Most Common Tech Hub Pairing)**
- Tokyo: 9 AM - 6 PM JST (UTC+9)
- San Francisco: 9 AM - 6 PM PST (UTC-8)
- Overlap: San Francisco 5 PM - 6 PM + Tokyo 10 AM - 11 AM = 1 hour
- OR Tokyo 4 AM - 6 PM + San Francisco 1 PM - 3 PM = 2 hours (evening for Tokyo)
- Reality: Minimal overlap, async-first essential

**Singapore + New York**
- Singapore: 9 AM - 6 PM SGT (UTC+8)
- New York: 9 AM - 6 PM EST (UTC-5)
- Overlap: Singapore 10 PM - 2 AM next day + New York 8 AM - 12 PM = 4 hours
- Best window: 1 PM - 2 PM New York time (10 PM Singapore)
- Strategy: Singapore team works late or rotates early morning shifts

**Sydney + London (Surprisingly Good Overlap)**
- Sydney: 9 AM - 6 PM AEDT (UTC+11, during summer)
- London: 9 AM - 6 PM GMT (UTC+0)
- Overlap: Sydney 8 PM - 6 AM next day + London 9 AM - 6 PM = 8 hours (if Sydney team starts at 4 PM)
- Best window: 3 PM - 4 PM London (1 AM - 2 AM Sydney next day)
- Strategy: Excellent for rotation; Sydney often has 4-5 hours of mid-morning overlap with London afternoon

**Bangalore + San Francisco**
- Bangalore: 9:30 AM - 6:30 PM IST (UTC+5:30)
- San Francisco: 9 AM - 6 PM PST (UTC-8)
- Overlap: Bangalore 11 PM - 6:30 AM + San Francisco 8 AM - 3 PM = 7.5 hours
- Best window: 8 AM - 9 AM San Francisco (11 PM - 12 AM Bangalore)
- Strategy: Bangalore evening aligns with San Francisco morning; good for handoffs

**Dubai + Europe (Good Sync, Growing Hub)**
- Dubai: 9 AM - 6 PM GST (UTC+4)
- London: 9 AM - 6 PM GMT (UTC+0)
- Overlap: 1 PM - 6 PM London + 5 PM - 10 PM Dubai = 5 hours shared afternoon
- Reality: Excellent overlap for core hours scheduling
- Strategy: Schedule core meetings 1 PM - 4 PM London = 5 PM - 8 PM Dubai

## Implementing Overlap Calculations in Production Code

For teams building custom scheduling or timezone tools, here's a more robust implementation handling edge cases:

```python
from datetime import datetime, timedelta
import pytz
from typing import List, Tuple

class TeamOverlapCalculator:
    def __init__(self):
        self.teams = {}

    def add_team(self, name: str, timezone: str, start_hour: int, end_hour: int):
        """Register a team with timezone and working hours."""
        self.teams[name] = {
            'tz': pytz.timezone(timezone),
            'start': start_hour,
            'end': end_hour
        }

    def find_overlap_windows(self, dates: int = 7) -> List[Tuple[str, float]]:
        """
        Find all overlap windows for next N days.
        Returns list of (time_window_description, overlap_hours)
        """
        overlaps = []

        for day_offset in range(dates):
            check_date = datetime.now() + timedelta(days=day_offset)

            # Convert all team working hours to UTC
            team_schedules = {}
            for team_name, config in self.teams.items():
                start = check_date.replace(hour=config['start'])
                end = check_date.replace(hour=config['end'])

                # Localize to team's timezone, then convert to UTC
                localized_start = config['tz'].localize(start)
                localized_end = config['tz'].localize(end)

                utc_start = localized_start.astimezone(pytz.UTC)
                utc_end = localized_end.astimezone(pytz.UTC)

                team_schedules[team_name] = (utc_start, utc_end)

            # Find UTC overlap
            utc_overlap_start = max(s[0] for s in team_schedules.values())
            utc_overlap_end = min(s[1] for s in team_schedules.values())

            if utc_overlap_start < utc_overlap_end:
                overlap_hours = (utc_overlap_end - utc_overlap_start).total_seconds() / 3600
                window_desc = f"{utc_overlap_start.strftime('%Y-%m-%d %H:%M UTC')} - {utc_overlap_end.strftime('%H:%M UTC')}"
                overlaps.append((window_desc, overlap_hours))

        return overlaps

    def get_meeting_time_options(self, max_options: int = 3) -> List[str]:
        """
        Suggest best meeting times that work for all teams.
        Returns times in each team's local timezone.
        """
        for day_offset in range(7):
            windows = self.find_overlap_windows(day_offset)
            if windows and len(windows) > 0:
                utc_time = windows[0][0].split(' ')[1]  # Extract UTC time
                suggestions = []

                for team_name, config in self.teams.items():
                    utc_dt = datetime.strptime(utc_time, '%H:%M').replace(tzinfo=pytz.UTC)
                    local_time = utc_dt.astimezone(config['tz']).strftime('%I:%M %p %Z')
                    suggestions.append(f"{team_name}: {local_time}")

                return suggestions

        return ["No full overlap found; consider splitting teams or async approach"]

# Usage Example
calc = TeamOverlapCalculator()
calc.add_team("Tokyo", "Asia/Tokyo", 9, 18)
calc.add_team("Austin", "America/Chicago", 8, 17)
calc.add_team("London", "Europe/London", 9, 18)

overlaps = calc.find_overlap_windows(7)
for window, hours in overlaps:
    print(f"{window}: {hours:.1f} hours of overlap")
```

## Scheduling Tools That Handle Timezone Complexity

Rather than building your own, consider these tools that automate overlap calculations:

- **Timezone.io** ($0 free) — Web-based timezone meeting planner, shows everyone's local time simultaneously
- **World Time Buddy** ($10/mo or free web version) — Visual grid showing all team members' current time
- **Calendar.com** ($10-20/mo) — Scheduling assistant that shows availability across timezones
- **Calendly Pro** ($16/mo) — Includes timezone-aware scheduling and overlap visualization
- **Google Calendar** ($0 if you have Google Workspace) — Add all teams' calendars; use color-coding to spot overlaps visually

## Measuring Success: Assessing Your Overlap Strategy

Track these metrics to evaluate whether your timezone strategy is working:

- Synchronous meeting count: Are you holding too many sync meetings despite minimal overlap? (Target: 1-2 per week for distributed teams)
- Async-to-sync ratio: Are team members able to accomplish work asynchronously? (Target: 80% async, 20% sync)
- Synchronous meeting attendance: Are all teams participating, or is one timezone always sacrificing sleep? (Target: balanced burden)
- Decision velocity: Are decisions getting made quickly in async, or are they stalling waiting for sync meetings?
- Employee satisfaction: Survey team members on whether the overlap strategy feels fair

If overlaps feel unfair (one timezone always working evening hours), rotate scheduled meeting times across weeks. If overlap is minimal but syncing wastes time, shift to async-first with brief async-recorded decision syncs.


## Related Articles

- [How to Calculate Productive Overlap Hours for Remote.](/remote-work-tools/how-to-calculate-productive-overlap-hours-for-remote-pair-pr/)
- [Calculate reasonable response windows based on overlap](/remote-work-tools/how-to-create-remote-team-communication-playbook-for-new-man/)
- [Example: Calculate optimal announcement time for global team](/remote-work-tools/how-to-communicate-remote-work-policy-changes-to-distributed/)
- [Remote Employee Time Zone Overlap Optimization Tool for](/remote-work-tools/remote-employee-time-zone-overlap-optimization-tool-for-scheduling-team-meetings/)
- [How to Manage Timezone Overlap When Working Remotely from](/remote-work-tools/how-to-manage-timezone-overlap-when-working-remotely-from-so/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
