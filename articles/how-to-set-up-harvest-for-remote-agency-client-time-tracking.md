---
layout: default
title: "How to Set Up Harvest for Remote Agency Client Time Tracking"
description: "A practical guide for setting up Harvest time tracking for remote agencies. Configure projects, set up client billing rates, and automate reporting."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-harvest-for-remote-agency-client-time-tracking/
categories: [guides]
tags: [harvest, time-tracking, remote-work, agency-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# How to Set Up Harvest for Remote Agency Client Time Tracking

Remote agencies face unique challenges when tracking time across distributed teams and multiple clients. Harvest provides a solution for capturing billable hours, managing client projects, and generating invoices. This guide covers practical setup steps for agencies working with remote clients.

## Creating Your Agency Workspace

Start by creating a Harvest account tailored to agency operations. The workspace structure determines how you organize client work and report on productivity.

When setting up, choose between a single workspace or multiple workspaces. Single workspace works well for agencies managing all clients in one place. Multiple workspaces suit larger agencies separating different business units or regional operations.

After account creation, invite team members through the team settings. Assign appropriate permission levels:

- Full Access: Can track time, create projects, manage invoices
- Light Access: Can track time and view assigned projects only 
- Time Only: Can only log hours with no project management access

For remote teams, ensure everyone downloads the mobile app for time tracking on the go. The browser extension provides one-click tracking from any webpage.

## Setting Up Clients and Projects

Client setup forms the foundation of your time tracking structure. Each client should have:

- Contact information for billing
- Default billing currency
- Hourly rate or fixed price agreement
- Communication preferences for invoices

Create projects under each client with clear naming conventions. Use a consistent format like `CLIENT-001 Project Name` for easy sorting and reporting. This helps when generating client reports or filtering by project in Harvest.

Configure these project-level settings:

```javascript
// Project configuration example
{
  "project_name": "Acme Corp Website Redesign",
  "client": "Acme Corporation",
  "billing_type": "Hourly",
  "hourly_rate": 150,
  "budget_method": "By project",
  "budget_hours": 200,
  "task_assignment": [
    { "task": "Discovery", "budget_hours": 20 },
    { "task": "Design", "budget_hours": 60 },
    { "task": "Development", "budget_hours": 80 },
    { "task": "QA & Testing", "budget_hours": 40 }
  ]
}
```

Task budgets help teams understand how much time remains for each project phase. Review these regularly during sprint planning or weekly check-ins.

## Configuring Hourly Rates and Billing

Agencies typically manage multiple rate structures: internal team member rates, client-facing rates, and potentially different rates for specific project phases.

Set up staff member hourly rates in the team settings. These rates calculate internal labor costs and help determine project profitability. The difference between internal rates and client billing rates represents gross margin.

Harvest supports several billing scenarios:

Hourly Rate Per Project: Charge a flat hourly rate for all work on a specific project. Use this when scope remains fluid and you bill for actual hours.

Task-Based Rates: Assign different rates to different task types. Design work might bill at $175/hour while development rates are $150/hour. Configure this in project settings under task assignments.

Fixed Fee Projects: For defined scope work, set a fixed price. Track time against the project while Harvest calculates earned value versus actual time spent.

Retainer Billing: Set up recurring invoices for ongoing client work. Track time against retainer projects, and Harvest applies hours against the prepaid amount.

## Time Tracking Workflows for Remote Teams

Establish clear time tracking habits that work across time zones. The key is consistency rather than complex processes.

Daily Tracking: Have team members log time at the end of each day. This prevents forgotten hours and keeps project budgets accurate. The Harvest timer works well for active work sessions.

Weekly Review: Designate a time weekly to review logged hours for accuracy. Team leads can run the "Team Overview" report to identify missing entries or suspicious patterns.

Code Snippet for Time Entry API: For teams wanting programmatic time tracking, Harvest provides a REST API:

```bash
# Create time entry via Harvest API
curl -X POST "https://api.harvestapp.com/v2/time_entries" \
  -H "Authorization: Bearer $HARVEST_ACCESS_TOKEN" \
  -H "Harvest-Account-Id: $ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": 12345678,
    "task_id": 123456,
    "spent_date": "2026-03-16",
    "hours": 4.5,
    "notes": "Implemented user authentication module"
  }'
```

Integrate this with your development workflow using GitHub Actions or a custom Slack command for seamless time logging without leaving your workflow.

## Generating Reports and Invoices

Harvest reporting helps agencies understand profitability, forecast workload, and bill clients accurately.

Project Profitability Report: Shows revenue versus costs for each project. Critical for understanding which clients and project types generate positive margins.

Budget vs Actual Report: Compares planned hours against logged time. Use this to identify projects heading over budget and initiate scope conversations with clients early.

Team Utilization Report: Tracks how much of available capacity your team is billing. Healthy agency utilization typically falls between 60-75% accounting for non-billable work like meetings and admin.

For client invoicing, create invoice templates with your agency branding. Include these elements:

- Itemized time entries with descriptions
- Task categories for clarity
- Payment terms and accepted methods
- Project reference numbers

Send invoices directly from Harvest or export to your accounting software. The integration with QuickBooks and Xero simplifies financial reconciliation.

## Integrating with Project Management Tools

Connect Harvest with your existing project management stack for streamlined workflows.

Slack Integration: Post time tracking reminders and weekly summaries to team channels. Configure notifications for missing time entries or budget alerts.

GitHub Integration: Link commits to Harvest time entries using the Harvest GitHub Actions workflow:

```yaml
name: Log Time to Harvest
on:
  push:
    branches: [main, develop]

jobs:
  log-time:
    runs-on: ubuntu-latest
    steps:
      - name: Create time entry
        run: |
          curl -X POST "https://api.harvestapp.com/v2/time_entries" \
            -H "Authorization: Bearer ${{ secrets.HARVEST_TOKEN }}" \
            -H "Harvest-Account-Id: ${{ secrets.ACCOUNT_ID }}" \
            -H "Content-Type: application/json" \
            -d '{
              "project_id": ${{ secrets.PROJECT_ID }},
              "task_id": ${{ secrets.TASK_ID }},
              "spent_date": "${{ github.event.head_commit.timestamp }}",
              "hours": 1.0,
              "notes": "Commit: ${{ github.sha }}"
            }'
```

API Webhooks: Set up webhooks to trigger actions when projects reach certain budget thresholds or when invoices are paid.

## Best Practices for Remote Agency Time Tracking

Implement these practices to maintain accurate time records:

1. Track time daily: Waiting until Friday means forgetting details from Monday through Thursday.

2. Write descriptive notes: Client-facing invoice descriptions should mean something. "Debugging" is less helpful than "Fixed login timeout issue on production server."

3. Use task budgets: They create accountability and early warning systems for scope creep.

4. Review utilization weekly: Catch underutilization before it becomes a problem.

5. Separate billable from non-billable: Track all time, but distinguish between client work and internal projects.

## Automating Administrative Tasks

Reduce manual overhead with Harvest's automation features:

- Recurring invoices: Schedule monthly invoices for retainer clients
- Timesheet reminders: Configure email reminders for missing daily entries 
- Budget alerts: Get notified when projects reach configurable threshold percentages

Set up these automations in the Settings > Notifications section. Tailor thresholds based on project size—smaller projects might warrant 75% alerts while larger engagements use 90%.

---


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
