---
layout: default
title: "Get recent workflow run durations"
description: "A technical guide for measuring and analyzing build times to identify developer productivity bottlenecks in remote engineering teams. Includes CI/CD"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /remote-engineering-team-build-time-tracking-as-developer-pro/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Build times directly impact developer productivity. When a remote engineering team waits 30 minutes for a CI pipeline to complete, that's 30 minutes of lost focus, context switching, and frustrated developers. Tracking build times systematically helps identify bottlenecks, optimize workflows, and measure the real impact of tooling decisions on team velocity.

## Table of Contents

- [Why Build Time Tracking Matters for Remote Teams](#why-build-time-tracking-matters-for-remote-teams)
- [Collecting Build Time Data](#collecting-build-time-data)
- [Analyzing Build Time Trends](#analyzing-build-time-trends)
- [Common Build Time Bottlenecks](#common-build-time-bottlenecks)
- [Setting Up Alerts](#setting-up-alerts)
- [Measuring Productivity Impact](#measuring-productivity-impact)
- [Comparing CI Platforms for Remote Teams](#comparing-ci-platforms-for-remote-teams)
- [Visualizing Build Time Data Over Time](#visualizing-build-time-data-over-time)
- [Establishing Build Time SLOs](#establishing-build-time-slos)
- [Sharing Build Time Reports with Non-Technical Stakeholders](#sharing-build-time-reports-with-non-technical-stakeholders)
- [Actionable Recommendations](#actionable-recommendations)
- [Related Reading](#related-reading)

This guide covers practical approaches to measuring, analyzing, and acting on build time data for distributed engineering teams.

## Why Build Time Tracking Matters for Remote Teams

Remote developers already face unique challenges: timezone coordination, async communication delays, and reduced spontaneous collaboration. Slow builds amplify these problems. A developer in Tokyo waiting for a CI pipeline that was optimized for a team in San Francisco faces compounded delays.

Build time tracking provides concrete data to:

- Identify which projects or services have the slowest CI times
- Measure the impact of dependency bloat on build speed
- Compare build times across different machine configurations
- Justify infrastructure investments with measurable data

## Collecting Build Time Data

Most CI platforms expose build duration through their APIs or logs. Here's how to collect this data from common platforms.

### GitHub Actions

GitHub Actions provides build duration in the workflow run details. You can extract this using the GitHub CLI:

```bash
# Get recent workflow run durations
gh run list --limit 20 --json durationMs,name,conclusion
```

This returns JSON with duration in milliseconds. Parse it to extract average build times per workflow.

### GitLab CI

GitLab exposes pipeline durations through the API:

```python
import requests

GITLAB_TOKEN = "your-token"
PROJECT_ID = "12345"

def get_pipeline_durations(project_id, max_pipelines=20):
    url = f"https://gitlab.com/api/v4/projects/{project_id}/pipelines"
    headers = {"Private-Token": GITLAB_TOKEN}
    params = {"per_page": max_pipelines}

    response = requests.get(url, headers=headers, params=params)
    pipelines = response.json()

    return [
        {
            "id": p["id"],
            "duration": p.get("duration"),  # seconds
            "created_at": p["created_at"],
            "status": p["status"]
        }
        for p in pipelines if p.get("duration")
    ]
```

### CircleCI

CircleCI provides build metadata through their V2 API:

```bash
curl -H "Circle-Token: $CIRCLECI_TOKEN" \
  "https://circleci.com/api/v2/project/gh/org/repo/pipeline?page=1&per_page=30"
```

## Analyzing Build Time Trends

Raw duration data needs context. A 10-minute build might be slow for a small service but fast for a monorepo with hundreds of packages. Here's a Python script to analyze trends and identify anomalies:

```python
from datetime import datetime, timedelta
from collections import defaultdict
import statistics

class BuildTimeAnalyzer:
    def __init__(self, build_data):
        self.builds = build_data

    def average_by_workflow(self):
        """Group builds by workflow name and calculate averages."""
        workflows = defaultdict(list)

        for build in self.builds:
            workflow = build.get("workflow", "unknown")
            duration = build.get("duration", 0)
            if duration > 0:
                workflows[workflow].append(duration)

        return {
            name: {
                "avg_seconds": statistics.mean(durations),
                "p95_seconds": sorted(durations)[int(len(durations) * 0.95)]
                                if len(durations) > 20 else None,
                "sample_size": len(durations)
            }
            for name, durations in workflows.items()
        }

    def detect_regression(self, threshold_seconds=60):
        """Find builds that exceed threshold, suggesting regression."""
        regressions = []

        for build in self.builds:
            if build.get("duration", 0) > threshold_seconds:
                regressions.append({
                    "workflow": build.get("workflow"),
                    "duration": build.get("duration"),
                    "date": build.get("date"),
                    "commit": build.get("commit")
                })

        return sorted(regressions, key=lambda x: x["duration"], reverse=True)
```

## Common Build Time Bottlenecks

Once you have data, you'll likely discover similar patterns across most teams:

### 1. Dependency Resolution

npm, pip, and Maven downloads during CI runs add significant latency. Cache dependencies aggressively:

```yaml
# GitHub Actions example
- name: Cache node modules
  uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

### 2. Test Suite Execution

Parallelizing tests dramatically reduces build times. Use tools like pytest-xdist for Python:

```bash
# Run tests in 4 parallel processes
pytest -n 4
```

For JavaScript projects, jest supports parallel execution by default. Configure maxWorkers:

```javascript
// jest.config.js
module.exports = {
  maxWorkers: "50%",
  // ... other config
};
```

### 3. Container Image Builds

Multi-stage Docker builds and layer caching help:

```dockerfile
# Bad: Every layer changes
FROM node:18
COPY . .
RUN npm install
RUN npm run build

# Good: Dependencies cached separately
FROM node:18 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18 AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./
COPY . .
RUN npm run build
```

## Setting Up Alerts

Don't wait for developers to complain about slow builds. Set up automated alerts:

```python
import subprocess
from datetime import datetime

def check_build_health():
    """Check if latest builds exceed threshold."""
    builds = get_recent_builds()  # Your API call here

    threshold_seconds = 300  # 5 minutes

    slow_builds = [
        b for b in builds
        if b["duration"] > threshold_seconds
    ]

    if slow_builds:
        message = f"⚠️ {len(slow_builds)} builds exceeded {threshold_seconds}s threshold"
        notify_slack(message)  # Your notification function
```

## Measuring Productivity Impact

Translate build times into developer hours lost:

```python
def calculate_cost_of_slow_builds(avg_build_time_minutes, builds_per_day, developers):
    daily_build_hours = (avg_build_time_minutes * builds_per_day) / 60
    annual_cost = daily_build_hours * 230 * 50  # 230 work days, $50/hour

    return {
        "daily_hours_lost": daily_build_hours,
        "annual_cost_usd": annual_cost,
        "recommendation": "Investigate build optimization"
                         if annual_cost > 5000 else "Acceptable"
    }
```

If your team runs 50 builds per day averaging 10 minutes each, that's over 8 hours daily—translating to roughly $100,000 annually in lost developer time.

## Comparing CI Platforms for Remote Teams

Not all CI platforms perform equally across geographies. For distributed teams, runner location and concurrency limits matter as much as raw build speed. Here is how the major platforms compare on dimensions relevant to remote engineering:

| Platform | Runner Regions | Concurrent Jobs (free) | Build Minutes (free/mo) | Self-hosted Option |
|---|---|---|---|---|
| GitHub Actions | 10+ regions | 20 (free tier) | 2,000 | Yes |
| GitLab CI | Multi-region | Varies by tier | 400 | Yes |
| CircleCI | US/EU/AP | 1 (free tier) | 6,000 | Yes |
| Buildkite | Self-hosted | Unlimited | N/A | Required |
| Depot | US/EU | 2 (free tier) | 500 | No |

For a remote team spread across Americas and Asia-Pacific, GitHub Actions or CircleCI with multiple runner pools reduces the latency that developers in distant time zones experience when waiting on pipeline results. Buildkite with self-hosted runners in each region is the most aggressive option when build speed is a first-class engineering priority.

## Visualizing Build Time Data Over Time

Raw numbers in a terminal are hard to act on. Push build metrics into a dashboard that your whole team can monitor. Grafana with a Prometheus data source works well for self-hosted solutions:

```yaml
# prometheus.yml scrape config for a custom build metrics exporter
scrape_configs:
  - job_name: 'build-times'
    static_configs:
      - targets: ['localhost:9100']
    metrics_path: '/metrics'
    scrape_interval: 5m
```

Expose metrics from your collection script:

```python
from prometheus_client import Gauge, start_http_server

build_duration_gauge = Gauge(
    'ci_build_duration_seconds',
    'CI build duration in seconds',
    ['workflow', 'branch', 'status']
)

def update_metrics(builds):
    for build in builds:
        build_duration_gauge.labels(
            workflow=build['workflow'],
            branch=build['branch'],
            status=build['status']
        ).set(build['duration'])

start_http_server(9100)
```

Even a simple shared Google Sheet updated weekly with average build times by service creates accountability and makes regressions visible before they compound. The key is making the data visible to the whole engineering organization, not just the team that owns the CI configuration.

## Establishing Build Time SLOs

Service Level Objectives work for build times just as they do for production APIs. Setting a formal SLO—say, 95% of builds complete within 8 minutes—creates a shared standard the team owns together.

Define your SLO in a simple document accessible to all engineers:

- **Target**: P95 build time under 8 minutes for the main pipeline
- **Measurement window**: Rolling 7 days
- **Alert threshold**: P95 exceeds 10 minutes for 2 consecutive days
- **Review cadence**: Monthly engineering sync, 5-minute agenda item

When a new dependency, test, or Docker layer causes a regression, the SLO makes it immediately clear that action is required. Without a formal target, slow build creep goes unnoticed until developers start complaining informally—a much harder signal to act on from a remote management position.

## Sharing Build Time Reports with Non-Technical Stakeholders

Engineering managers at remote companies often need to communicate build time improvements to product or executive leadership who do not read dashboards. A simple weekly digest in Slack keeps the conversation grounded in data rather than anecdote.

Template for a weekly build health message:

```
*Build Health Report — Week of [DATE]*
• Avg CI time (main pipeline): 7m 42s (down from 9m 15s last week)
• Slowest workflow: integration-tests (avg 14m 30s)
• Total build minutes consumed: 12,400 (budget: 15,000)
• P95 within SLO (8m): YES

Action items: Investigate integration-test parallelization before next sprint.
```

Sending this in a dedicated #engineering-metrics channel every Monday takes five minutes and prevents the common pattern where build time regressions go unnoticed for weeks because no one thought to check.

## Actionable Recommendations

Start with quick wins:

1. **Enable dependency caching** in your CI configuration—typically saves 2-5 minutes per build
2. **Parallelize test execution**—can reduce test suite time by 60-80%
3. **Audit dependencies**—remove unused packages, upgrade to latest versions
4. **Consider build agents** closer to your developer locations for remote teams

Track build times weekly and set a team target of keeping average CI time under 10 minutes. Anything longer actively harms productivity and should be prioritized for optimization.

Build by theluckystrike — More at [zovo.one](https://zovo.one)

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

{% endraw %}

## Related Reading

- [Remote Team Support Ticket First Response Time Tracking for](/remote-work-tools/remote-team-support-ticket-first-response-time-tracking-for-/)
- [How to Run Book Clubs for a Remote Engineering Team of 40](/remote-work-tools/how-to-run-book-clubs-for-a-remote-engineering-team-of-40/)
- [How to Get Recurring Clients as a Freelance Developer](/remote-work-tools/how-to-get-recurring-clients-as-freelance-developer/)
- [Reading schedule generator for async book clubs](/remote-work-tools/how-to-run-async-book-clubs-for-distributed-engineering-teams/)
- [Example: GitHub Actions workflow for assessment tracking](/remote-work-tools/how-to-set-up-remote-hiring-pipeline-with-async-interviews-f/)

## Related Articles

- [GitHub Actions Workflow for Remote Dev Teams](/remote-work-tools/github-actions-remote-dev-workflow/)
- [How to Build Remote Team Documentation Culture Guide](/remote-work-tools/how-to-build-remote-team-documentation-culture-guide/)
- [How to Build Remote Team Async Culture from Scratch 2026](/remote-work-tools/how-to-build-remote-team-async-culture-from-scratch-2026/)
- [How to Build a Remote Team Handbook from Scratch](/remote-work-tools/how-to-build-a-remote-team-handbook-from-scratch/)
- [How to Build Trust on Fully Remote Teams](/remote-work-tools/how-to-build-trust-on-fully-remote-teams/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
