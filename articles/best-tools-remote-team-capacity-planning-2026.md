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

## Table of Contents

- [The Capacity Planning Model](#the-capacity-planning-model)
- [1. Linear (Best for Engineering Teams)](#1-linear-best-for-engineering-teams)
- [2. Notion Capacity Tracker](#2-notion-capacity-tracker)
- [3. Float (Best for Agencies and Multi-Project)](#3-float-best-for-agencies-and-multi-project)
- [4. GitHub Projects v2 with Capacity Fields](#4-github-projects-v2-with-capacity-fields)
- [5. Spreadsheet: Quarterly Headcount Capacity](#5-spreadsheet-quarterly-headcount-capacity)
- [Weekly Async Capacity Update Template](#weekly-async-capacity-update-template)
- [6. Choosing the Right Tool by Team Size](#6-choosing-the-right-tool-by-team-size)
- [7. Integrating Capacity Data into Your Async Workflow](#7-integrating-capacity-data-into-your-async-workflow)
- [8. Capacity Planning Anti-Patterns for Remote Teams](#8-capacity-planning-anti-patterns-for-remote-teams)
- [9. Quarterly Capacity Review Process](#9-quarterly-capacity-review-process)
- [Team Capacity Overview](#team-capacity-overview)
- [Allocation by Initiative](#allocation-by-initiative)
- [Risks](#risks)
- [Decisions Needed](#decisions-needed)
- [Related Reading](#related-reading)

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

## 6. Choosing the Right Tool by Team Size

Team size significantly shapes which capacity planning tool adds the most value without creating management overhead.

**Teams of 2-8 engineers** rarely need a dedicated capacity planning tool. A shared Notion database with the sprint capacity formula above, combined with a weekly async update in Slack, handles planning well. Linear's built-in cycle tracking plus the bash script covers sprint-level visibility without additional tooling cost.

**Teams of 8-25 engineers** are where Float and dedicated capacity tools earn their cost. At this size, cross-team dependencies become real: the frontend team's capacity affects the backend team's ability to ship, and capacity shortfalls in infrastructure slow down product delivery. A visual timeline in Float or a multi-team view in Linear makes these dependencies visible before they become blockers.

**Teams of 25+ engineers** usually need both sprint-level and quarterly capacity planning running in parallel. Engineering managers track sprint capacity in Linear or Jira; engineering directors track quarterly headcount capacity in Float or a spreadsheet model. The quarterly model informs hiring decisions and program commitments. The sprint model informs day-to-day prioritization.

## 7. Integrating Capacity Data into Your Async Workflow

For remote teams, the most important capacity planning insight is available to team members when they need it — not locked in a manager's spreadsheet or discussed only in a meeting that happened at 2am local time.

A practical setup that works for many remote engineering teams:

```bash
# Automated capacity summary posted to Slack every Monday
# Uses Linear API to pull current cycle data

#!/bin/bash
CYCLE_DATA=$(curl -s -X POST "https://api.linear.app/graphql" \
  -H "Authorization: Bearer $LINEAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query { cycles(filter: {isActive: {eq: true}}) { nodes { name issues { nodes { estimate assignee { name } state { name } } } } } }"}')

# Generate summary
SUMMARY=$(echo "$CYCLE_DATA" | jq -r '
  .data.cycles.nodes[0].issues.nodes |
  group_by(.assignee.name) |
  map({
    person: .[0].assignee.name,
    planned: map(.estimate // 0) | add,
    done: map(select(.state.name == "Done") | .estimate // 0) | add
  }) |
  .[] |
  "\(.person): \(.done)/\(.planned) pts complete"
' | tr '\n' '\n')

# Post to Slack
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"*Sprint Capacity Update*\n\`\`\`\n${SUMMARY}\n\`\`\`\"}"
```

This automation posts to a `#capacity-planning` Slack channel every Monday morning, giving the whole team visibility into workload distribution without anyone having to pull the data manually or attend a standup to hear it.

## 8. Capacity Planning Anti-Patterns for Remote Teams

The tools above solve the visibility problem, but capacity planning fails for reasons that are not tool-related.

**Ignoring meeting overhead.** Engineering teams typically estimate capacity in story points but forget to subtract time spent in planning, retrospective, design reviews, and 1:1s. A 10-day sprint for a senior engineer who runs three recurring meetings per week may have only 7-8 days of effective engineering capacity. The bash calculator above handles this with the `meeting_overhead` parameter, but it only works if managers enter realistic values rather than aspirational ones.

**Planning to 100% allocation.** Teams planned at full capacity have no buffer for the unexpected: production incidents, urgent bug fixes, or scope that expands mid-sprint. Experienced engineering managers plan to 80% of calculated capacity and treat the buffer as insurance. Remote teams should be more conservative — distributed coordination has higher latency, which means unexpected issues take longer to resolve.

**Not tracking actuals against estimates.** Capacity planning that does not close the feedback loop is guesswork. Track story points planned versus completed each sprint. If a team consistently completes 70% of planned capacity, the model needs adjustment — either estimates are too optimistic or the capacity calculation is missing overhead.

## 9. Quarterly Capacity Review Process

Beyond sprint-level planning, remote engineering teams benefit from a quarterly capacity review that takes a longer view. A lightweight async process that works:

```markdown
# Q2 Capacity Review — Async Template
**Due: April 1, post in Notion before EOD**

## Team Capacity Overview
- Total team-days in Q2: [run quarterly-capacity.py]
- Committed projects: [list with rough effort estimates]
- Hiring plan: [headcount changes expected]
- Known absences: [team PTO, parental leave, etc.]

## Allocation by Initiative
| Initiative | Estimated Days | Assigned Team Members |
|-----------|---------------|----------------------|
| Core product roadmap | | |
| Technical debt | | |
| On-call and reliability | | |
| Cross-team dependencies | | |
| Buffer (20%) | | |

## Risks
- [List any capacity risks: hiring delays, scope uncertainty, dependencies]

## Decisions Needed
- [Surface any allocation tradeoffs that need manager or leadership input]
```

This document gets posted to Notion and reviewed asynchronously by engineering managers and product leadership. Decisions are documented in the same thread rather than in a meeting that half the remote team cannot attend at a convenient time.

## Related Reading

- [Async Capacity Planning Process for Remote Engineering Managers](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-managers-guide/)
- [How to Coordinate Remote SRE Team Capacity Planning](/remote-work-tools/how-to-coordinate-remote-sre-team-capacity-planning-across-i/)
- [Best Tools for Remote Team Metrics Dashboards](/remote-work-tools/best-tools-remote-team-metrics-dashboards/)
- [Best Tool for Remote Team Capacity Planning When Scaling](/remote-work-tools/best-tool-for-remote-team-capacity-planning-when-scaling-eng/)

---

## Related Articles

- [Best Tools for Remote Team Capacity Planning in 2026](/remote-work-tools/best-tools-for-remote-team-capacity-planning-2026/)
- [Best Tool for Remote Team Capacity Planning When Scaling](/remote-work-tools/best-tool-for-remote-team-capacity-planning-when-scaling-eng/)
- [Async Capacity Planning Process for Remote Engineering](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-manag/)
- [Async Capacity Planning Process for Remote: Managers](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-managers-guide/)
- [Best Tools for Remote Team Sprint Planning (2026)](/remote-work-tools/best-tools-for-remote-team-sprint-planning-2026/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
