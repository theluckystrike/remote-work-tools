---
layout: default
title: "Example: Add a client to a specific project list"
description: "A technical guide to configuring ClickUp client portals for remote project visibility, with API examples, automation scripts, and best practices for."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-clickup-client-portal-for-remote-project-visib/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
---


To set up a ClickUp client portal, create a dedicated space with guest access configured to specific lists, then use ClickUp's API to automate guest provisioning and filter client-facing views. This approach gives remote development teams visibility into project progress while keeping internal technical discussions private.

## Guest Access vs. Client Portal: Understanding Your Options

ClickUp offers two primary mechanisms for external client visibility:

1. **Guest Access** — Invite clients as guests to specific spaces, folders, or lists. Guests receive credentials but can only see what you explicitly share.
2. **Client Portal** — Available on Business and Enterprise plans, this provides a white-labeled, polished interface that looks less like internal project management.

For most development teams, guest access provides sufficient functionality and works across all plan tiers. Here's how to implement it programmatically.

## Setting Up Guest Access via API

While you can create guests through the ClickUp UI, automating guest provisioning fits better into developer workflows. Here's a Python script using the ClickUp API:

```python
import os
import requests

CLICKUP_API_KEY = os.getenv("CLICKUP_API_KEY")
TEAM_ID = os.getenv("CLICKUP_TEAM_ID")

def create_client_guest(email, name, accessible_list_ids):
    """Create a guest user with access to specific lists."""
    url = f"https://api.clickup.com/api/v2/team/{TEAM_ID}/guest"
    
    payload = {
        "email": email,
        "name": name,
        "can_see_time": True,
        "list_ids": accessible_list_ids
    }
    
    headers = {
        "Authorization": CLICKUP_API_KEY,
        "Content-Type": "application/json"
    }
    
    response = requests.post(url, json=payload, headers=headers)
    return response.json()

# Example: Add a client to a specific project list
To set up a ClickUp client portal, create a dedicated space with guest access configured to specific lists, then use ClickUp's API to automate guest provisioning and filter client-facing views. This approach gives remote development teams visibility into project progress while keeping internal technical discussions private.

This approach works well when you need to provision multiple clients across different projects—simply extend the `accessible_list_ids` array to match your project structure.

## Structuring Client-Facing Spaces

Create a dedicated space structure that separates client-visible content from internal development work. A practical folder layout looks like:

```
Client Projects/
├── Acme Corp Website/
│ ├── 01_Project_Plan (Client View)
│ ├── 02_Milestones (Client View)
│ ├── 03_Deliverables (Client View)
│ └── Internal_Discussions (Team Only)
```

The key principle: curate spaces explicitly for clients rather than exposing your entire workspace. Clients should see milestones, deliverables, and status—not sprint planning, bug backlogs, or internal code review discussions.

## Custom Views for Client Visibility

Configure custom views that filter out technical details. Use ClickUp's view API to create client-specific perspectives:

```javascript
// ClickUp API: Create a filtered view for clients
const createClientView = async (listId) => {
 const response = await fetch(`https://api.clickup.com/api/v2/list/${listId}/view`, {
 method: "POST",
 headers: {
 "Authorization": process.env.CLICKUP_API_KEY,
 "Content-Type": "application/json"
 },
 body: JSON.stringify({
 "name": "Client Progress View",
 "filters": {
 "status": ["Not Started", "In Progress", "Complete"],
 "assignees": [] // Show all tasks
 },
 "filter_version": 2,
 "show_subtasks": true,
 "visible_fields": ["name", "due_date", "status", "assignees", "attachments"]
 })
 });

 return response.json();
};
```

This view includes only task names, due dates, status, assignees, and attachments—stripping out custom fields that might contain cost data, internal priority markers, or technical notes.

## Automation Patterns for Client Updates

Automate status updates to reduce manual communication overhead. This Integromat/Make scenario sends weekly summaries to clients:

```javascript
// Webhook payload handler for weekly client digest
const generateClientDigest = async (clientEmail, projectId) => {
 // Fetch completed tasks from the past week
 const tasks = await clickup.getTasks({
 list_id: projectId,
 filter: {
 statuses: ["complete"],
 date_updated: {
 start: weekAgo(),
 end: now()
 }
 }
 });

 // Format the digest
 const completed = tasks.filter(t => t.status.status === "complete");
 const inProgress = tasks.filter(t => t.status.status === "in_progress");

 return {
 to: clientEmail,
 subject: `Project Update: ${completed.length} tasks completed this week`,
 body: `
 Completed: ${completed.map(t => t.name).join(", ")}
 In Progress: ${inProgress.map(t => t.name).join(", ")}

 View full details: ${dashboardUrl}
 `
 };
};
```

You can also set up automation within ClickUp itself:

- **Task Complete → Notify Client**: When a task status changes to "Complete," automatically add a comment visible to the client guest
- **Blocker Added → Alert Manager**: Notify your project lead when a client-dependent task is blocked
- **Due Date Passed → Escalate**: Route overdue items awaiting client feedback to your account manager

## Integrating with External Dashboards

For clients who prefer a custom dashboard outside ClickUp, pull data via the API:

```python
from clickup_api import ClickUpClient
import json

def export_project_status(space_id):
 """Export project status as JSON for external dashboards."""
 client = ClickUpClient(api_key=os.getenv("CLICKUP_API_KEY"))

 # Get all lists in the space
 lists = client.get_lists(space_id)

 status_data = {
 "project_name": space_id,
 "milestones": [],
 "tasks_by_status": {
 "pending": 0,
 "in_progress": 0,
 "complete": 0
 }
 }

 for lst in lists:
 tasks = client.get_tasks(lst.id)
 for task in tasks:
 status = task.status.status.lower().replace(" ", "_")
 if status in status_data["tasks_by_status"]:
 status_data["tasks_by_status"][status] += 1

 return status_data

# Serve via Flask for client dashboard
@app.route("/api/project-status")
def project_status():
 return jsonify(export_project_status("acme_website"))
```

This pattern works well when you need to embed project status into a client portal running on your own domain.

## Permission Auditing for Security

Periodically audit guest permissions to prevent accidental exposure:

```python
def audit_guest_access():
 """List all guests and their accessible resources."""
 client = ClickUpClient(api_key=os.getenv("CLICKUP_API_KEY"))

 team_members = client.get_team_members()
 guests = [m for m in team_members if m.get("is_guest")]

 audit_report = []
 for guest in guests:
 guest_id = guest["id"]
 accessible = client.get_guest_sharedFolders(guest_id)

 audit_report.append({
 "email": guest["email"],
 "name": guest["name"],
 "accessible_folders": [f["name"] for f in accessible],
 "last_active": guest.get("last_active")
 })

 return audit_report
```

Run this monthly to ensure former clients no longer have access and current clients only see what they need.

## Practical Tips for Developer Teams

- **Use descriptive task names**: Clients see task titles directly. Instead of `FEAT-142`, use "Implement user authentication flow"
- **Set up separate notification rules**: Guests should only receive mentions on tasks they're assigned to, not every comment
- **Create client-specific templates**: Build task templates for deliverables that prompt for client-facing descriptions
- **Document the setup**: Keep internal docs explaining which spaces are client-accessible so new team members don't accidentally share wrong content

The client portal setup is not a one-time configuration—treat it as part of your client service infrastructure that evolves based on feedback and usage patterns.


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [How to Set Up Client Onboarding Portal for Remote Agency](/remote-work-tools/how-to-set-up-client-onboarding-portal-for-remote-agency/)
- [How to Set Up Basecamp for Remote Agency Client.](/remote-work-tools/how-to-set-up-basecamp-for-remote-agency-client-communicatio/)
- [How to Set Up HubSpot for Remote Agency Client Pipeline](/remote-work-tools/how-to-set-up-hubspot-for-remote-agency-client-pipeline/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
