---
layout: default
title: "Get recent workflow run durations"
description: "A technical guide for measuring and analyzing build times to identify developer productivity bottlenecks in remote engineering teams. Includes CI/CD."
date: 2026-03-16
author: theluckystrike
permalink: /remote-engineering-team-build-time-tracking-as-developer-pro/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Build times directly impact developer productivity. When a remote engineering team waits 30 minutes for a CI pipeline to complete, that's 30 minutes of lost focus, context switching, and frustrated developers. Tracking build times systematically helps identify bottlenecks, optimize workflows, and measure the real impact of tooling decisions on team velocity.

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

## Actionable Recommendations

Start with quick wins:

1. **Enable dependency caching** in your CI configuration—typically saves 2-5 minutes per build
2. **Parallelize test execution**—can reduce test suite time by 60-80%
3. **Audit dependencies**—remove unused packages, upgrade to latest versions
4. **Consider build agents** closer to your developer locations for remote teams

Track build times weekly and set a team目标 of keeping average CI time under 10 minutes. Anything longer actively harms productivity and should be prioritized for optimization.

Build by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}

## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [Remote Team Support Ticket First Response Time Tracking.](/remote-work-tools/remote-team-support-ticket-first-response-time-tracking-for-/)
- [How to Manage Work-Life Balance as a Remote Developer](/remote-work-tools/how-to-manage-work-life-balance-remote-developer/)
- [Remote Team Story Point Velocity Trend Analysis Tool for.](/remote-work-tools/remote-team-story-point-velocity-trend-analysis-tool-for-sprint-planning-guide/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
