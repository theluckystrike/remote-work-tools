---
layout: default
title: ".GitHub/ISSUE_TEMPLATE/oncall-shift.md"
description: "A practical guide for developers and power users comparing tools and methods to track and balance on-call burden across distributed remote teams"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /best-tool-for-tracking-remote-team-on-call-burden-distributi/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, best-of, remote-work]---

{% raw %}
## The Problem with Untracked On-Call Burden


Distributed engineering teams often treat on-call as an informal arrangement where engineers rotate through scheduled shifts and everyone assumes the load is roughly equal. It rarely is. One engineer working in a timezone that overlaps with the system's peak traffic hours absorbs significantly more incidents than someone in a quieter timezone. A senior engineer who owns legacy components catches more escalations than a junior team member. Without data, these imbalances are invisible until someone burns out or leaves.

Tracking on-call burden isn't about surveillance. It's about having the numbers to have a fair conversation. When you can show a teammate that they handled 14 incidents last month while the median was 6, adjusting their next rotation is an obvious call. Without the data, that conversation becomes subjective and uncomfortable.

This guide covers the tools and approaches that work best for remote distributed teams—from teams running everything through GitHub to those using dedicated incident management platforms.


## Key Takeaways

- **PagerDuty Business tier (approximately**: $41/user/month as of 2026) unlocks the full analytics suite including time-of-day breakdowns, which is where the real fairness data lives.
- **This guide covers the**: tools and approaches that work best for remote distributed teams—from teams running everything through GitHub to those using dedicated incident management platforms.
- **Require shift reports as**: non-optional documentation after each rotation 3.
- **Set explicit thresholds**: define what constitutes an overloaded shift before problems emerge

The "best" tool ultimately depends on what you already have.
- **Teams with PagerDuty can**: use existing analytics.

## Starting with GitHub Issues: Low Overhead, High Visibility

For teams already using GitHub, the fastest path to burden tracking is a shift report template stored in `.github/ISSUE_TEMPLATE/oncall-shift.md`. Each engineer opens an issue at the start of their shift and closes it when handing off. The issue captures incidents, sleep impact, and general notes.

A practical template:

```markdown---
name: On-Call Shift Report
about: Document your on-call shift for burden tracking
title: "[ON-CALL] [Engineer Name] - [Date Range]"
labels: oncall
---

## Shift Summary

**Engineer:**
**Start:**
**End:**

## Incidents Handled

| # | Time | Severity | Title | Resolution Time |
|---|------|----------|-------|------------------|
| 1 | | | | |

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
 --template '{{range.}}{{.number}} {{.title}} by {{.assignee.login}} on {{.created}}{{"\n"}}{{end}}'
```

This gives you raw incident counts per person, though it doesn't capture severity or resolution time. For teams with fewer than 8 engineers and low incident volumes, this is often enough. The data lives in GitHub where everyone already works, there's no additional tool to maintain, and the audit trail is permanent.

The limitation shows when incident volumes grow. Searching issues and manually aggregating counts stops being practical above roughly 50 incidents per month. At that point, you need purpose-built analytics.

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

PagerDuty Business tier (approximately $41/user/month as of 2026) unlocks the full analytics suite including time-of-day breakdowns, which is where the real fairness data lives. If your team already pays for PagerDuty, extracting burden reports is a configuration exercise. If you're evaluating it for the first time, factor the analytics tier cost into the decision.

## OpsGenie as a PagerDuty Alternative

Atlassian's OpsGenie offers comparable on-call management at a lower price point, making it common among teams already using Jira and Confluence. Its reporting module surfaces:

- On-call time per engineer per schedule
- Alert response time distributions
- Escalation frequency

The Atlassian integration means on-call burden data can appear alongside sprint metrics in Jira dashboards, giving engineering managers a consolidated view without exporting reports between tools.

For teams in the Atlassian ecosystem, OpsGenie's schedule integration with Jira automation allows incident tickets to auto-assign based on who is currently on-call, reducing context-switching and ensuring the burden record follows the engineer through resolution.

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
 preferred_hours: tuple # (start_hour, end_hour)

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

This simple approach can be extended to integrate with actual scheduling tools via webhook or API. Teams with more complex requirements—multiple services, shadow on-call arrangements, or cross-functional coverage—benefit from adding severity weighting. A P1 incident at 2 AM should count more than a P3 alert at 11 AM, and the rotation algorithm should reflect that.

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

Grafana On-Call makes particular sense for teams that are already running Grafana for observability. The alert routing integrates directly with Prometheus AlertManager rules, so alerts flow into the same on-call system that tracks who responded. This creates a closed loop: fire an alert, route it to the on-call engineer, record the response, feed it back into burden tracking—all within a single platform.

The trade-off is setup time. Unlike PagerDuty or OpsGenie, Grafana On-Call requires meaningful configuration effort before it produces useful burden reports.

## Comparing Tools Side by Side

| Tool | Free Tier Analytics | Night Incident Tracking | API Access | Best For |
|------|--------------------|-----------------------|------------|----------|
| GitHub Issues | Manual only | No | Yes | Small teams, low volume |
| PagerDuty | No (paid required) | Yes | Yes | Mid-to-large teams |
| OpsGenie | Limited | Yes | Yes | Atlassian ecosystem teams |
| Grafana On-Call | Yes | Partial | Yes | Open-source infrastructure teams |
| Custom spreadsheet | Manual | Manual | N/A | Any team starting out |

No single tool is best for all teams. The right choice depends on what you already pay for, how large your team is, and whether you need analytics that justify the cost to finance.

## Key Metrics to Track Monthly

Regardless of tool choice, track these metrics monthly to ensure fair burden distribution:

1. **Incidents acknowledged** - Raw count per person
2. **Incidents resolved** - Distinguishes responders from acknowledgers
3. **Night incidents (12 AM - 6 AM)** - High-burden events
4. **Total on-call hours** - Includes scheduled but quiet shifts
5. **Post-incident follow-up time** - Investigation and documentation work
6. **Escalation rate** - How often the primary on-call escalates to a secondary

Create a simple spreadsheet or dashboard to visualize these numbers. If one engineer consistently appears in the top quartile for night incidents across multiple months, that's a signal to adjust rotation priority.

## Handling Time Zone Fairness for Distributed Teams

On-call burden in a globally distributed team has an invisible layer of unfairness baked into standard rotation schedules. An engineer in Bangalore covering an US product's on-call shift is absorbing incidents during their sleep hours. An engineer in Berlin covering the same rotation may handle those same incidents during their afternoon.

Equal incident counts across a rotation don't mean equal burden when time zones are involved. Address this by:

- Segmenting incident data by local time of day for each engineer, not UTC
- Creating separate shift types: business hours primary, business hours secondary, after-hours primary
- Compensating after-hours shifts with shorter shift duration or rotation priority credits

Some teams formalize this as a "burden score" where a night incident counts as 3 points, an evening incident as 2, and a business-hours incident as 1. Monthly burden scores replace raw incident counts as the fairness metric.

## Practical Steps to Implement Today

Start tracking on-call burden without purchasing new tools:

1. **Create a shared spreadsheet** with columns for engineer, month, incidents, night incidents, and resolution hours
2. **Require shift reports** as non-optional documentation after each rotation
3. **Review burden monthly** in team retrospectives
4. **Adjust upcoming schedules** based on previous month's data
5. **Set explicit thresholds** — define what constitutes an overloaded shift before problems emerge

The "best" tool ultimately depends on what you already have. Teams with GitHub can start immediately using issues. Teams with PagerDuty can use existing analytics. Teams running Kubernetes can adopt Grafana On-Call as a natural extension of their observability stack.

Fair on-call distribution is a solved problem at the tracking level—the challenge is consistently reviewing the data and actually adjusting rotations based on what it reveals.

## Frequently Asked Questions

**How often should teams review on-call burden data?**
Monthly is the minimum effective cadence. Weekly reviews in engineering retrospectives work well for teams with high incident volumes or a history of burnout concerns.

**What if engineers on different services have very different incident rates?**
Track burden per service, not just per team. An engineer owning a legacy payment processor may carry 10x the burden of someone owning a reporting microservice. Cross-service rotation—where senior engineers cycle through high-burden services—is one solution.

**Should quiet on-call shifts count in burden calculations?**
Yes. Being available and on standby carries a psychological cost even when no incidents fire. Count scheduled on-call hours regardless of incident volume, and consider them alongside incident counts.

**How do you handle on-call compensation fairly for distributed teams?**
Some organizations pay an on-call stipend per shift. Others offer compensatory time off after high-burden periods. Document the policy explicitly and apply it consistently—ambiguous policies create more resentment than the burden itself.

## Related Articles

- [.github/ISSUE_TEMPLATE/onboarding.yml](/remote-work-tools/hybrid-team-onboarding-process-template-for-new-hires-splitting-time-office-and-home/)
- [.github/workflows/conflict-escalation.yaml](/remote-work-tools/remote-team-conflict-resolution-over-chat-when-video-call-is/)
- [Shortcut vs Linear: Issue Tracking Comparison for](/remote-work-tools/shortcut-vs-linear-issue-tracking-comparison/)
- [Example: GitHub Actions workflow for assessment tracking](/remote-work-tools/how-to-set-up-remote-hiring-pipeline-with-async-interviews-f/)
- [Example GitHub PR template](/remote-work-tools/how-to-transition-from-sync-meetings-to-async-updates-gradua/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

