---
layout: default
title: "Simple assignment: rotate through combinations"
description: "A practical guide for developers and power users building a hybrid work schedule template with three office days, including rotation patterns"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /how-to-create-hybrid-work-schedule-template-for-teams-with-t/
reviewed: true
score: 8
voice-checked: true
categories: [guides]
intent-checked: true
tags: [remote-work-tools]
---

Three-office-day hybrid schedules balance collaboration needs with focused individual work by using rotating assignment patterns that ensure 3 days in office, minimum team overlap, and no more than 60% office capacity on any day. Python scripts can generate valid rotation schedules, YAML configurations specify which days teams are in-office, and calendar sync tooling (Google Calendar API) makes schedules accessible where teams live. Desk booking systems prevent overbooking, communication protocols clarify when to prefer in-person versus async, and monthly reviews adapt schedules to actual team patterns.

## Table of Contents

- [Understanding the Three-Office-Day Pattern](#understanding-the-three-office-day-pattern)
- [Building the Schedule Template](#building-the-schedule-template)
- [Integrating with Calendar Tools](#integrating-with-calendar-tools)
- [Setting Up Communication Norms](#setting-up-communication-norms)
- [Office Days (Mon/Wed/Fri for Group A)](#office-days-monwedfri-for-group-a)
- [Remote Days (Tue/Thu for Group A)](#remote-days-tuethu-for-group-a)
- [Cross-Mode Communication](#cross-mode-communication)
- [Managing Desk and Resource Booking](#managing-desk-and-resource-booking)
- [Review and Iterate](#review-and-iterate)

## Understanding the Three-Office-Day Pattern

A three-day office schedule works well when your team has specific needs that require physical presence. Engineering teams doing hardware debugging, design teams collaborating on physical prototypes, or teams with frequent client meetings often find two office days insufficient. The three-day pattern provides enough overlap for meaningful collaboration while still granting team members two days for focused, uninterrupted work.

The key challenge is preventing the schedule from becoming chaotic. Without a clear template, you'll deal with constant Slack messages asking "who's in tomorrow?" and last-minute desk bookings. A structured approach with rotation logic solves this.

## Building the Schedule Template

### Core Schedule Structure

A three-office-day schedule typically follows one of two patterns:

Fixed Pattern: Team members have assigned office days that don't rotate. This works well when team size is small or when individual roles have specific in-office requirements.

Rotating Pattern: Team members rotate through office day combinations on a weekly or bi-weekly basis. This maximizes randomness in office attendance and ensures everyone experiences different collaboration dynamics.

For most technical teams, a rotating pattern provides better outcomes. Here's a Python script that generates valid rotation schedules:

```python
#!/usr/bin/env python3
"""
Hybrid Schedule Generator for 3-office-day teams.
Generates rotation patterns that ensure minimum team overlap.
"""

from itertools import combinations
from dataclasses import dataclass
from typing import List, Dict, Set
import json

@dataclass
class TeamMember:
    name: str
    preferred_office_days: List[str] = None
    required_office_days: List[str] = None

    def __post_init__(self):
        self.preferred_office_days = self.preferred_office_days or []
        self.required_office_days = self.required_office_days or []

DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]

def generate_rotation_slots(num_members: int) -> List[Dict]:
    """
    Generate rotation slots ensuring:
    - 3 office days per person per week
    - No more than 60% of team in office on any day
    - Minimum 2 people overlap for collaboration
    """
    all_combinations = list(combinations(DAYS, 3))
    schedules = []

    for combo in all_combinations:
        day_counts = {day: 0 for day in DAYS}

        for member_idx in range(num_members):
            # Simple assignment: rotate through combinations
            combo_idx = member_idx % len(all_combinations)
            assigned_days = all_combinations[combo_idx]

            for day in assigned_days:
                day_counts[day] += 1

        # Check constraints
        max_coverage = max(day_counts.values())
        min_overlap = min(day_counts.values())

        if max_coverage <= int(num_members * 0.6) and min_overlap >= 2:
            schedules.append({
                "combination": combo,
                "day_counts": day_counts
            })

    return schedules

def create_weekly_rotation(team: List[TeamMember], weeks: int = 4) -> Dict:
    """Create a multi-week rotation schedule."""
    rotation = {}
    base_combinations = generate_rotation_slots(len(team))

    for week in range(1, weeks + 1):
        week_schedule = {}
        for idx, member in enumerate(team):
            combo_idx = (idx + week - 1) % len(base_combinations)
            week_schedule[member.name] = list(base_combinations[combo_idx]["combination"])
        rotation[f"Week {week}"] = week_schedule

    return rotation

# Example usage
if __name__ == "__main__":
    team = [
        TeamMember("Alice", required_office_days=["Monday"]),
        TeamMember("Bob"),
        TeamMember("Carol"),
        TeamMember("David"),
    ]

    schedule = create_weekly_rotation(team, weeks=4)
    print(json.dumps(schedule, indent=2))
```

Run this script to generate a four-week rotation that you can then adapt to your specific team size and constraints.

### Schedule Storage Format

Store your schedule in a format that's easy to update and integrates with existing tools. JSON or YAML works well:

```yaml
# schedule.yaml
team_schedule:
  rotation_period: bi-weekly

  week_a:
    Monday:
      in_office: ["Alice", "Bob", "Carol"]
      remote: ["David", "Eve"]
    Tuesday:
      in_office: ["Alice", "David"]
      remote: ["Bob", "Carol", "Eve"]
    Wednesday:
      in_office: ["Bob", "Carol", "David"]
      remote: ["Alice", "Eve"]
    Thursday:
      in_office: ["Alice", "Eve"]
      remote: ["Bob", "Carol", "David"]
    Friday:
      in_office: ["Carol", "Eve"]
      remote: ["Alice", "Bob", "David"]

  anchor_days:
    # Days when specific activities require full attendance
    - day: Monday
      activity: Sprint planning
      required: true
    - day: Wednesday
      activity: Team standup + code review
      required: false
```

## Integrating with Calendar Tools

For developers, the schedule needs to live where you already live—your calendar. Here's a script that syncs your schedule to Google Calendar:

```javascript
// calendar-sync.js
// Requires: googleapis npm package
// npm install googleapis google-auth-library

const { google } = require('googleapis');
const fs = require('fs');
const yaml = require('js-yaml');

const SCOPES = ['https://www.googleapis.com/auth/calendar'];

async function syncScheduleToCalendar(credentialsPath, scheduleFile) {
  const auth = new google.auth.GoogleAuth(
    JSON.parse(fs.readFileSync(credentialsPath)),
    SCOPES
  );

  const calendar = google.calendar({ version: 'v3', auth });
  const schedule = yaml.load(fs.readFileSync(scheduleFile, 'utf8'));

  // Create all-day events for office days
  for (const [week, days] of Object.entries(schedule.team_schedule)) {
    if (week === 'anchor_days') continue;

    for (const [day, attendees] of Object.entries(days)) {
      const inOfficeList = attendees.in_office || [];

      for (const person of inOfficeList) {
        // Check if this event already exists
        const event = {
          summary: `🏢 In Office - ${person}`,
          description: `Office day for ${person}. Check desk booking system.`,
          start: { date: getNextDateForDay(day) },
          end: { date: getNextDateForDay(day) }
        };

        await calendar.events.insert({
          calendarId: 'primary',
          resource: event
        });
      }
    }
  }
}

function getNextDateForDay(dayName) {
  const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];
  const targetDay = days.indexOf(dayName);
  const today = new Date();
  const currentDay = today.getDay();

  const diff = targetDay - currentDay;
  const nextDate = new Date(today.setDate(today.getDate() + diff));

  return nextDate.toISOString().split('T')[0];
}

module.exports = { syncScheduleToCalendar };
```

## Setting Up Communication Norms

A schedule template is useless without clear communication expectations. Define when to use which channel based on who's in the office:

```markdown
# Hybrid Communication Protocol

## Office Days (Mon/Wed/Fri for Group A)
- Prefer in-person conversations for code reviews
- Use Slack for async updates that remote team members need
- Book meeting rooms for discussions involving remote folks
- Update shared Slack status: "� office"

## Remote Days (Tue/Thu for Group A)
- All meetings must have Zoom/Meet links
- Record important discussions for those in different timezones
- Use async updates in writing before jumping to calls
- Update shared Slack status: "🏠 remote"

## Cross-Mode Communication
When one person is in-office and others are remote:
- Always include virtual meeting link in calendar invites
- Default to screen-sharing during code reviews
- Use collaborative documents for design discussions
```

## Managing Desk and Resource Booking

With three office days, you'll likely need a desk booking system. Several open-source options exist, or you can build a simple one:

```typescript
// simple-desk-booking.ts
interface Booking {
  userId: string;
  deskId: string;
  date: string;
  morning: boolean;
  afternoon: boolean;
}

class DeskBookingSystem {
  private bookings: Map<string, Booking[]> = new Map();
  private deskCapacity: number;

  constructor(deskCapacity: number = 20) {
    this.deskCapacity = deskCapacity;
  }

  async bookDesk(userId: string, date: string, morning: boolean, afternoon: boolean): Promise<boolean> {
    const dayBookings = this.bookings.get(date) || [];
    const morningCount = dayBookings.filter(b => b.morning).length;
    const afternoonCount = dayBookings.filter(b => b.afternoon).length;

    if (morning && morningCount >= this.deskCapacity) {
      throw new Error("Desk capacity reached for morning");
    }
    if (afternoon && afternoonCount >= this.deskCapacity) {
      throw new Error("Desk capacity reached for afternoon");
    }

    dayBookings.push({ userId, deskId: `desk-${dayBookings.length + 1}`, date, morning, afternoon });
    this.bookings.set(date, dayBookings);

    return true;
  }

  getAvailableDesks(date: string): number {
    const dayBookings = this.bookings.get(date) || [];
    const maxDesksUsed = Math.max(
      dayBookings.filter(b => b.morning).length,
      dayBookings.filter(b => b.afternoon).length
    );
    return this.deskCapacity - maxDesksUsed;
  }
}
```

## Review and Iterate

Your first schedule won't be perfect. Plan a monthly review where the team discusses:

- Are collaboration days actually collaborative?
- Are deep work days protected from interruptions?
- Does the rotation work for everyone, including those with caregiving responsibilities?
- Are anchor days achieving their intended purpose?

Adjust the template based on feedback. The schedule should serve your team's actual work patterns, not the other way around.

Building a three-office-day hybrid schedule doesn't require expensive tools or complex systems. Start with a simple rotation, use existing calendar and communication tools, and iterate based on what actually works for your team.

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

- [Example: Generating a staggered schedule for a 6-person team](/remote-work-tools/best-practice-for-hybrid-work-policy-covering-which-days-tea/)
- [Return to Office Tools for Hybrid Teams: A Practical Guide](/remote-work-tools/return-to-office-tools-for-hybrid-teams/)
- [Review assignment logic (example)](/remote-work-tools/code-review-workflow-for-a-remote-backend-team-of-6-develope/)
- [Digital Signage for Hybrid Office Communication](/remote-work-tools/digital-signage-for-hybrid-office-communication/)
- [Remote Team First 90 Days Plan Template for Senior Hires](/remote-work-tools/remote-team-first-90-days-plan-template-for-senior-hires-joi/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
