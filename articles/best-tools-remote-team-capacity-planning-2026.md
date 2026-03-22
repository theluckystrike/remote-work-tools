---
layout: default
title: "Best Tools for Remote Team Capacity Planning"
description: "Top tools remote engineering managers use for capacity planning across sprints, quarters, and headcount with async-friendly visibility into workload"
date: 2026-03-22
author: theluckystrike
permalink: /best-tools-remote-team-capacity-planning-2026/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Capacity planning for remote teams is harder than in-office: you can't glance across the office to see who is overloaded. The tools that work surface workload data without requiring managers to chase status updates. This guide covers the best options by team type, plus the spreadsheet formulas and automations that tie them together.

## Key Takeaways

- **This guide covers the**: best options by team type, plus the spreadsheet formulas and automations that tie them together.
- **Topics covered**: the capacity planning model, 1. linear (best for engineering teams), 2. notion capacity tracker
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## The Capacity Planning Model

```
Capacity = Available days × Focus ratio

Example:
  Engineer: 10 days in sprint
  Minus: 1 day meetings + admin
  Minus: 0.5 days on-call rotation
  Minus: 0.5 days PTO
  = 8 days effective capacity

  Story point velocity × 8/10 = adjusted sprint capacity
```

```bash
# Sprint capacity calculator (bash)
#!/bin/bash
calculate_capacity() {
  local name="$1"
  local sprint_days="$2"
  local pto_days="${3:-0}"
  local oncall_days="${4:-0}"
  local meeting_overhead="${5:-0.1}"  # 10% default

  local available=$(echo "$sprint_days - $pto_days - $oncall_days" | bc)
  local capacity=$(echo "scale=1; $available * (1 - $meeting_overhead)" | bc)
  echo "$name: ${capacity} effective days (from ${sprint_days} sprint days, -${pto_days} PTO, -${oncall_days} on-call)"
}

calculate_capacity "Alice"   10 1 0.5 0.15
calculate_capacity "Bob"     10 0 0   0.10
calculate_capacity "Carlos"  10 2 0   0.12
```

## 1. Linear (Best for Engineering Teams)

**Cost:** $8/user/month
**Best for:** Sprint-based engineering with cycle tracking

```bash
# Linear API: get current cycle capacity
curl -X POST "https://api.linear.app/graphql" \
  -H "Authorization: Bearer $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query {
      cycles(filter: {isActive: {eq: true}}) {
        nodes {
          name
          startsAt
          endsAt
          issues {
            nodes {
              title
              estimate
              state { name }
              assignee { name }
            }
          }
        }
      }
    }"
  }' | jq '
    .data.cycles.nodes[0].issues.nodes |
    group_by(.assignee.name) |
    map({
      person: .[0].assignee.name,
      total_points: map(.estimate // 0) | add,
      open_issues: map(select(.state.name != "Done")) | length
    })
  '
```

Linear capacity dashboard setup:

```
Team Settings > Cycles
  Cycle length: 2 weeks
  Default story point scale: Fibonacci (1,2,3,5,8,13)

Member settings:
  Alice:   10 pts/cycle capacity
  Bob:     10 pts/cycle capacity
  Carlos:  8 pts/cycle (30% on other team)

Auto-limit: Warn when member exceeds their capacity target
```

## 2. Notion Capacity Tracker

For teams already in Notion, a database works well:

```markdown
# Sprint Capacity Database Properties:

Sprint (Select): Sprint 42
Engineer (Person)
Sprint Days (Number)
PTO Days (Number)
On-Call Days (Number)
Meeting Overhead % (Number)
Story Points Planned (Number)
Story Points Completed (Number)

# Formulas:
Available Days = Sprint Days - PTO Days - On-Call Days
Effective Capacity = Available Days * (1 - Meeting Overhead % / 100)
% Allocated = Story Points Planned / (Effective Capacity * 2)
# Where 2 = assumed story points per day

Status (Formula):
  if(prop("% Allocated") < 0.8, "Under",
  if(prop("% Allocated") < 1.1, "On Track", "Overloaded"))
```

## 3. Float (Best for Agencies and Multi-Project)

**Cost:** $6/user/month
**Best for:** Teams working across multiple projects with billable hour tracking

```bash
# Float API: get team availability for next 2 weeks
curl "https://api.float.com/v3/people" \
  -H "Authorization: Bearer $FLOAT_API_TOKEN" | \
  jq '.[] | {name: .name, department: .department, hours_per_day: .work_days.hours}'

# Check utilization
curl "https://api.float.com/v3/tasks?start_date=$(date +%Y-%m-%d)&end_date=$(date -d '+14 days' +%Y-%m-%d)" \
  -H "Authorization: Bearer $FLOAT_API_TOKEN" | \
  jq '
    group_by(.people_id) |
    map({
      person_id: .[0].people_id,
      total_hours: map(.hours) | add
    })
  '
```

## 4. GitHub Projects v2 with Capacity Fields

Add custom capacity fields to GitHub Projects:

```bash
# Create custom field via API
gh api graphql -f query='
  mutation {
    addProjectV2Field(input: {
      projectId: "PVT_xxx",
      dataType: NUMBER,
      name: "Story Points"
    }) {
      projectV2Field {
        ... on ProjectV2Field {
          id
          name
        }
      }
    }
  }
'

# Get sprint workload by assignee
gh api graphql -f query='
  query {
    organization(login: "yourorg") {
      projectV2(number: 1) {
        items(first: 100) {
          nodes {
            content {
              ... on Issue {
                title
                assignees(first: 1) { nodes { login } }
              }
            }
            fieldValues(first: 10) {
              nodes {
                ... on ProjectV2ItemFieldNumberValue {
                  number
                  field { ... on ProjectV2Field { name } }
                }
              }
            }
          }
        }
      }
    }
  }
' | jq '
  .data.organization.projectV2.items.nodes |
  map({
    title: .content.title,
    assignee: (.content.assignees.nodes[0].login // "unassigned"),
    points: (.fieldValues.nodes[] | select(.field.name == "Story Points") | .number) // 0
  }) |
  group_by(.assignee) |
  map({
    person: .[0].assignee,
    total_points: map(.points) | add
  })
'
```

## 5. Spreadsheet: Quarterly Headcount Capacity

For quarterly planning, a Google Sheets model:

```
# Quarterly Capacity Model (CSV format)

Name,Role,Jan,Feb,Mar,Q1 Capacity (days)
Alice,Backend,19,18,21,58
Bob,Backend,21,20,20,61
Carlos,Frontend,19,18,15,52
Diana,Frontend,21,20,21,62
Total,,,,233

Working days minus:
  Company holidays: -2 per month = -6 per person per quarter
  Sick/admin buffer: -2 per quarter
  Effective Q1: 233 - (4 * 8) = 201 team-days
```

```python
# scripts/quarterly-capacity.py
from datetime import date, timedelta
import holidays
import json

def working_days(year, month):
    us_holidays = holidays.US(years=year)
    count = 0
    d = date(year, month, 1)
    while d.month == month:
        if d.weekday() < 5 and d not in us_holidays:
            count += 1
        d += timedelta(days=1)
    return count

def quarterly_capacity(team, year, quarter):
    months = {1: [1,2,3], 2: [4,5,6], 3: [7,8,9], 4: [10,11,12]}
    q_months = months[quarter]

    total_days = sum(working_days(year, m) for m in q_months)

    result = []
    for member in team:
        pto = member.get('pto_days', 0)
        overhead = member.get('overhead_pct', 0.1)
        capacity = (total_days - pto) * (1 - overhead)
        result.append({
            'name': member['name'],
            'raw_days': total_days,
            'pto': pto,
            'effective_days': round(capacity, 1)
        })

    return result

team = [
    {'name': 'Alice', 'pto_days': 5, 'overhead_pct': 0.12},
    {'name': 'Bob', 'pto_days': 3, 'overhead_pct': 0.10},
    {'name': 'Carlos', 'pto_days': 10, 'overhead_pct': 0.15},
]

capacity = quarterly_capacity(team, 2026, 2)
print(json.dumps(capacity, indent=2))
# Total team capacity: sum of effective_days
```

## Weekly Async Capacity Update Template

```markdown
# Capacity Update — Week of March 23

**Post in:** #capacity-planning by Monday 10am

Format:
@[name]: [available days this week] days | Focus: [primary project]

Example:
@alice: 4 days (1 day on-call) | Focus: Auth service refactor
@bob: 3 days (2 days PTO Fri-Sat) | Focus: CI pipeline
@carlos: 5 days | Focus: Frontend component library + design system review
```

## Related Reading

- [Async Capacity Planning Process for Remote Engineering Managers](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-managers-guide/)
- [How to Coordinate Remote SRE Team Capacity Planning](/remote-work-tools/how-to-coordinate-remote-sre-team-capacity-planning-across-i/)
- [Best Tools for Remote Team Metrics Dashboards](/remote-work-tools/best-tools-remote-team-metrics-dashboards/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
