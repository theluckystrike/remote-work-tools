---

layout: default
title: ".github/ISSUE_TEMPLATE/oncall-shift.md"
description: "A practical guide for developers and power users comparing tools and methods to track and balance on-call burden across distributed remote teams."
date: 2026-03-16
author: theluckystrike
permalink: /best-tool-for-tracking-remote-team-on-call-burden-distributi/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Tracking on-call burden fairly across distributed teams requires more than just remembering who was last on rotation. When team members span multiple time zones, work different schedules, or have varying seniority levels, manual tracking breaks down. The right approach combines clear rotation schedules with metrics that capture actual on-call burden, not just scheduled shifts.

## What Actually Constitutes On-Call Burden

On-call burden extends beyond the hours spent on rotation. True burden includes:

- Incidents woken up during the night
- Time spent resolving escalated issues
- After-hours interrupts that fragment rest
- Cognitive load carrying a pager affects next-day productivity

A developer who responds to three critical incidents at 3 AM carries a heavier burden than someone who sleeps through their entire shift, even if both were "on call" for the same hours. Fair distribution means accounting for this reality.

## Building a Custom On-Call Tracker with GitHub Issues

For teams already using GitHub, tracking on-call rotations through issues provides flexibility without additional tooling. Create a rotating issue template:

```yaml
# .github/ISSUE_TEMPLATE/oncall-shift.md
---
name: On-Call Shift Report
about: Document your on-call shift activity
title: "On-Call: [NAME] - [DATE RANGE]"
labels: oncall
assignee: @username
---

## Shift Summary

**Time Zone:** 
**Shift Start:** 
**Shift End:** 

## Incidents Handled

| # | Time | Severity | Title | Resolution Time |
|---|------|----------|-------|------------------|
| 1 |      |          |       |                  |

## Notes

Any context for the team about incidents or issues?

## Sleep Quality Impact

- [ ] Woke during night
- [ ] Fragmented sleep (multiple small interrupts)
- [ ] Full night sleep
```

Query incident burden across the team using GitHub's search API:

```bash
gh search issues --repo org/infrastructure \
  --label oncall,incident \
  --created "2026-01-01..2026-03-01" \
  --json number,title,assignee,created \
  --template '{{range .}}{{.number}} {{.title}} by {{.assignee.login}} on {{.created}}{{"\n"}}{{end}}'
```

This gives you raw incident counts per person, though it doesn't capture severity or resolution time.

## Using PagerDuty for Built-in Analytics

PagerDuty provides native analytics for on-call tracking, making it a strong choice for teams needing minimal setup. The platform tracks:

- Total incidents acknowledged and resolved
- Average response time per responder
- Escalation policy adherence

Pull on-call analytics via PagerDuty's API:

```python
import requests
from datetime import datetime, timedelta

def get_oncall_burden(pd_api_key, start_date, end_date):
    url = "https://api.pagerduty.com/analytics/incidents"
    headers = {
        "Authorization": f"Token token={pd_api_key}",
        "Content-Type": "application/json"
    }
    params = {
        "time_zone": "UTC",
        "start": start_date,
        "end": end_date
    }
    
    response = requests.get(url, headers=headers, params=params)
    data = response.json()
    
    # Aggregate by responder
    burden = {}
    for incident in data.get("incidents", []):
        for responder in incident.get("acknowledged_by", []):
            responder_id = responder["id"]
            if responder_id not in burden:
                burden[responder_id] = {"count": 0, "total_minutes": 0}
            burden[responder_id]["count"] += 1
    
    return burden
```

The limitation with PagerDuty is that free tiers restrict analytics access, and the data focuses on incident counts rather than holistic burden including after-hours disruption to personal time.

## Building Fair Rotation Logic

Beyond tracking, proactively designing fair rotations requires considering factors beyond equal shift counts. Implement a rotation algorithm that weights by:

1. **Incident volume** - If someone handled more incidents last month, reduce their upcoming shifts
2. **Time zone coverage** - Ensure primary coverage during business hours for the team's main regions
3. **Seniority calibration** - Junior team members paired with seniors during on-call shifts
4. **Recovery time** - Mandatory rest period after night-time incidents

Example rotation scheduler in Python:

```python
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import List

@dataclass
class Engineer:
    id: str
    name: str
    incidents_last_30d: int
    timezone: str
    preferred_hours: tuple  # (start_hour, end_hour)

def calculate_shift_weight(engineer: Engineer) -> float:
    """Lower weight = more eligible for upcoming shift"""
    base_weight = 1.0
    
    # Penalize high incident volume
    incident_factor = 1 + (engineer.incidents_last_30d * 0.1)
    
    # Combine factors
    return base_weight * incident_factor

def suggest_next_oncall(engineers: List[Engineer]) -> str:
    """Suggest next on-call engineer based on fair distribution"""
    weights = {e.id: calculate_shift_weight(e) for e in engineers}
    
    # Return engineer with lowest burden weight
    selected_id = min(weights, key=weights.get)
    return next(e.name for e in engineers if e.id == selected_id)

# Example usage
team = [
    Engineer("e1", "Alex", 3, "UTC", (9, 17)),
    Engineer("e2", "Jordan", 7, "PST", (9, 17)),
    Engineer("e3", "Casey", 2, "EST", (9, 17)),
]

print(f"Next on-call: {suggest_next_oncall(team)}")
```

This simple approach can be extended to integrate with actual scheduling tools via webhook or API.

## Grafana On-Call for Open-Source Teams

For teams running on open-source infrastructure, Grafana On-Call provides a free option with scheduling, escalation, and notification management. It integrates with Prometheus for alert routing and offers basic analytics:

- Who was on-call when
- How many alerts fired
- Escalation chain usage

Export on-call data for custom analysis:

```bash
curl -X GET "https://grafana.example.com/api/oncall/v1/schedules" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json"
```

Parse the response to calculate coverage hours per engineer and identify imbalances.

## Key Metrics to Track Monthly

Regardless of tool choice, track these metrics monthly to ensure fair burden distribution:

1. **Incidents acknowledged** - Raw count per person
2. **Incidents resolved** - Distinguishes responders from acknowledgers
3. **Night incidents (12 AM - 6 AM)** - High-burden events
4. **Total on-call hours** - Includes scheduled but quiet shifts
5. **Post-incident follow-up time** - Investigation and documentation work

Create a simple spreadsheet or dashboard to visualize these numbers. If one engineer consistently appears in the top quartile for night incidents across multiple months, that's a signal to adjust rotation priority.

## Practical Steps to Implement Today

Start tracking on-call burden without purchasing new tools:

1. **Create a shared spreadsheet** with columns for engineer, month, incidents, night incidents, and resolution hours
2. **Require shift reports** as non-optional documentation after each rotation
3. **Review burden monthly** in team retrospectives
4. **Adjust upcoming schedules** based on previous month's data

The "best" tool ultimately depends on what you already have. Teams with GitHub can start immediately using issues. Teams withPagerDuty can use existing analytics. Teams running Kubernetes can adopt Grafana On-Call as a natural extension of their observability stack.

Fair on-call distribution is a solved problem at the tracking level—the challenge is consistently reviewing the data and actually adjusting rotations based on what it reveals.


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [Remote Team Penetration Testing Coordination Guide for.](/remote-work-tools/remote-team-penetration-testing-coordination-guide-for-distr/)
- [Remote Sales Team Commission Tracking Tool for.](/remote-work-tools/remote-sales-team-commission-tracking-tool-for-distributed-s/)
- [Remote Team Financial Dashboard Tool for CFO: Tracking.](/remote-work-tools/remote-team-financial-dashboard-tool-for-cfo-tracking-distri/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
