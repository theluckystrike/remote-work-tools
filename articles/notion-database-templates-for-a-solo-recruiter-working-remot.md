---
layout: default
title: "Notion Database Templates for a Solo Recruiter Working Remot"
description: "A practical guide to building custom Notion database templates for solo recruiters working remotely. Includes database schemas, formulas, and."
date: 2026-03-16
author: theluckystrike
permalink: /notion-database-templates-for-a-solo-recruiter-working-remot/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
Building a personal ATS (Applicant Tracking System) with Notion databases gives solo recruiters working remotely a powerful, customizable tool without enterprise software costs. Notion's relational database structure maps naturally to recruitment workflows, and this guide shows you how to construct templates that scale with your hiring volume.

## Why Notion Works for Solo Recruiters

Solo recruiters face unique challenges: managing multiple pipelines simultaneously, tracking communication across platforms, and maintaining candidate relationships without a dedicated ATS team. Notion solves this through three core features: relational databases, formula properties, and template buttons.

Unlike monolithic ATS platforms, Notion lets you design databases that match your exact workflow. You control the fields, views, and automation. The trade-off is that you build some functionality that enterprise tools provide out-of-the-box—but the flexibility rewards your investment.

## Core Database Architecture

A functional recruitment system requires three interconnected databases: Candidates, Companies, and Jobs. Here's how to structure each.

### Candidates Database

Create a database with these core properties:

| Property | Type | Purpose |
|----------|------|---------|
| Name | Title | Candidate name |
| Email | Email | Contact information |
| Phone | Phone | Alternative contact |
| Current Status | Select | Pipeline stage |
| Applied Date | Created time | Automatic timestamp |
| Source | Select | Where candidate found |
| Associated Job | Relation | Link to job database |
| Last Contact | Date | Follow-up tracking |
| Rating | Select | Evaluation score |

The `Current Status` select property should include stages like: New, Screening, Interview, Offer, Hired, Rejected, Withdrawn.

### Companies Database

Track companies separately to avoid data duplication:

| Property | Type | Purpose |
|----------|------|---------|
| Company Name | Title | Organization name |
| Industry | Multi-select | Sector classification |
| Size | Select | Employee count range |
| Location | Text | Headquarters |
| Open Jobs | Relation | Links to job postings |
| Primary Contact | Person | Internal hiring manager |
| Last Outreach | Date | Communication tracking |

### Jobs Database

The jobs database serves as your source of truth for all requisitions:

| Property | Type | Purpose |
|----------|------|---------|
| Job Title | Title | Position name |
| Company | Relation | Link to company |
| Department | Select | Team or division |
| Location | Select | Remote/hybrid/onsite |
| Salary Range | Text | Compensation band |
| Status | Status | Active/Paused/Closed |
| Candidates | Relation | Linked applicants |
| Posted Date | Date | When job went live |
| Time to Hire | Formula | Days from post to hire |

## Formula Examples for Automation

Notion formulas transform static databases into dynamic tracking systems. Here are practical formulas for recruitment workflows.

### Days Since Last Contact

```notion
dateBetween(now(), prop("Last Contact"), "days")
```

This formula calculates how many days have passed since your last candidate touchpoint. Use it in a filter to surface candidates needing follow-up:

```
Filter: Days Since Last Contact > 7
```

### Candidate Age

```notion
dateBetween(now(), prop("Applied Date"), "days")
```

Track how long candidates sit in your pipeline. Combine with status filters to identify bottlenecks:

```
Filter: Candidate Age > 14 AND Current Status = "Interview"
```

### Pipeline Conversion Rate

```notion
format(round(prop("Hired") / prop("Applied") * 100)) + "%"
```

Calculate your conversion from applied to hired. Requires rollup properties counting candidates per status.

## Template Button Workflows

Template buttons automate repetitive tasks. Create a button in your candidates database that:

1. Sets status to "New"
2. Creates a linked "Initial Outreach" task in a tasks database
3. Opens an email draft to the candidate

```javascript
// Notion API template button action (conceptual)
{
  "action": "createTask",
  "properties": {
    "name": `Initial Outreach - ${candidate.name}`,
    "dueDate": "today + 2 days",
    "relatedTo": candidate.id
  }
}
```

## View Configurations for Daily Use

Views determine what you see. Build multiple views for different contexts:

### My Candidates This Week

```
Filter: Assignee = "Me" AND Last Contact > 7 days ago
Sort: Last Contact ascending
```

### Hot Pipeline (Interview Stage)

```
Filter: Current Status = "Interview"
Sort: Rating descending
View: Board (grouped by Company)
```

### Follow-Up Needed

```
Filter: Last Contact < today() - 5 days
Show: Name, Company, Last Contact, Phone
```

## Integration with Communication Tools

Solo recruiters juggle email, Slack, and calendar. Connect Notion to these tools using native integrations or automation platforms:

Calendar Integration: Use Notion's calendar view for interviews. Sync with Google Calendar or Cal.com for candidate-facing scheduling.

Email Tracking: Notion doesn't track email opens natively. Use a separate email tool with a blind BCC to a personal inbox, then manually update "Last Contact" in Notion after significant exchanges.

Slack Reminders: Set up Slack reminders that query Notion:

```
/remind me "Follow up with candidates in interview stage" every Monday at 9am
```

## Scaling Your System

As your candidate volume grows, these patterns help maintain efficiency:

1. Use relations, not text fields: Linking candidates to jobs and companies enables powerful rollups and cross-database views.

2. Implement status automation: When a candidate moves to "Hired," automatically archive the job if all positions are filled using Notion's native automation.

3. Create template pages per job: Each job entry can contain a linked page with interview scorecards, evaluation criteria, and team feedback.

4. Separate warm from cold outreach: Maintain different databases or tags for proactive sourcing versus reactive applications.

## What to Avoid

Don't over-engineer your system on day one. Start with basic candidate and job tracking, then add complexity as your workflow reveals gaps. Many solo recruiters build elaborate templates they never use.

Avoid storing sensitive data like salary negotiations or internal feedback in databases shared with hiring managers. Use separate private databases for confidential information.

## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [Notion Setup for Solo Freelancer Managing 5 Clients: A Practical Guide](/remote-work-tools/notion-setup-for-solo-freelancer-managing-5-clients/)
- [Remote Onboarding Checklist for a Solo HR Manager Hiring 10](/remote-work-tools/remote-onboarding-checklist-for-a-solo-hr-manager-hiring-10/)
- [Best Onboarding Automation Workflow for Remote Companies Using Slack Bots and Notion Templates](/remote-work-tools/best-onboarding-automation-workflow-for-remote-companies-using-slack-bots-and-notion-templates/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
