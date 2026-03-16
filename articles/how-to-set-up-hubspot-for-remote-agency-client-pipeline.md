---

layout: default
title: "How to Set Up HubSpot for Remote Agency Client Pipeline"
description: "A practical technical guide to configuring HubSpot pipelines for remote agencies. Includes CRM setup, custom properties, automation workflows, and API integrations for developer-focused teams."
date: 2026-03-15
author: "Remote Work Tools Guide"
permalink: /how-to-set-up-hubspot-for-remote-agency-client-pipeline/
categories: [guides]
reviewed: true
score: 8
---


HubSpot provides a robust foundation for managing client relationships in remote agency environments. While the platform offers extensive out-of-the-box functionality, configuring it properly requires understanding your agency's specific workflow requirements. This guide walks through the technical setup process with attention to automation, custom properties, and integrations that power users can implement without relying on premium tiers.

## Pipeline Architecture for Agency Workflows

The default deal pipeline in HubSpot works, but remote agencies need custom stages that reflect their actual sales process. Start by navigating to **Settings > Objects > Deals > Pipelines** and create a pipeline that matches your agency's stages.

For a typical remote agency, the pipeline should include:

```yaml
Deal Stages:
  - Lead In:           # Initial contact from website/form
  - Discovery Call:    # Schedule of initial call
  - Proposal Sent:     # Custom proposal delivered
  - Revision Round:    # Client feedback incorporated
  - Contract Review:   # Legal/NDA review
  - Active Project:    # Currently working
  - Retainer:          # Ongoing monthly work
  - Closed Won:        # Completed successfully
  - Closed Lost:       # Did not convert
```

Each stage should have clear criteria your team understands. The transition from "Proposal Sent" to "Revision Round" should mean something specific—not just "client hasn't responded yet."

## Custom Properties That Matter

Standard HubSpot properties cover basic contact information, but remote agencies need additional data points. Create custom properties that capture information relevant to your operations.

Navigate to **Settings > Properties > Deals** and add these properties:

| Property Name | Type | Purpose |
|---------------|------|---------|
| Project Type | Dropdown | Web Dev, Design, Marketing, Consulting |
| Budget Range | Dropdown | <$5k, $5k-$15k, $15k-$50k, $50k+ |
| Timezone | Single-line text | For scheduling across regions |
| Team Lead | Owner | Who manages this client account |
| Remote Stack | Multi-checkbox | Tools client uses (Notion, Slack, Jira) |
| Communication Pref | Dropdown | Async, Sync, Hybrid |

The "Team Lead" property is particularly useful when multiple people handle client relationships. It ensures accountability even when team members work across different time zones.

## Automation Without Premium Tiers

HubSpot's automation capabilities exist at every tier, but the free tier has meaningful constraints. Focus on workflows that provide high value without requiring paid seats.

### Contact-Based Workflows

Create a workflow that assigns leads based on criteria:

```javascript
// Workflow enrollment trigger
Contact property "Lead Source" is equal to "Website"
AND Contact property "Budget Range" is greater than "$5k"
```

Then set the action to "Create task for owner" with a reminder to follow up within 48 hours. This ensures no lead falls through the cracks regardless of team timezone distribution.

### Deal-Based Workflows

Set up stage-based notifications:

```javascript
// When deal stage changes to "Proposal Sent"
- Send email to contact (using template with proposal link)
- Create task for owner: "Follow up in 3 days"
- Rotate through team members using round-robin (if multiple owners)
```

For remote agencies, email templates should include timezone-aware scheduling links. HubSpot's meeting scheduler integrates here, but you can also embed Calendly or Cal.com links if you prefer those tools.

## Form Integration for Lead Capture

Remote agencies typically generate leads through their website. HubSpot forms embed cleanly, but for developer-focused setups, the API provides more control.

Create a custom HTML form that submits to HubSpot's API:

```javascript
// Submit form data to HubSpot API
async function submitLead(formData) {
  const portalId = 'YOUR_PORTAL_ID';
  const formGuid = 'YOUR_FORM_GUID';
  
  const response = await fetch(
    `https://api.hsforms.com/submissions/v3/integration/submit/${portalId}/${formGuid}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        fields: [
          { name: 'email', value: formData.email },
          { name: 'firstname', value: formData.name },
          { name: 'company', value: formData.company },
          { name: 'project_type', value: formData.projectType }
        ],
        context: {
          pageUri: window.location.href,
          pageName: document.title
        }
      })
    }
  );
  
  return response.json();
}
```

This approach gives you full styling control while still populating your HubSpot CRM. The API submission also triggers workflows based on form responses.

## Reporting and Dashboard Configuration

Remote agencies need visibility into pipeline health without requiring everyone to log into HubSpot daily. Create a dashboard that surfaces the metrics that matter:

**Key Reports for Agency Owners:**
- Deal velocity (days in each stage)
- Win rate by project type
- Average deal size by source
- Pipeline value by owner

Build these in **Reports > Reports > Create Custom Report**. For a quick snapshot, the pipeline visualization shows where deals stall:

```yaml
Pipeline Health Check:
  - Leads in "Discovery Call" > 14 days: Review follow-up process
  - Deals in "Proposal Sent" > 30 days: Evaluate proposal effectiveness  
  - Conversion rate "Active Project" → "Retainer": Measure upsell success
```

Share dashboards via email on a weekly cadence. Remote teams appreciate async updates rather than live dashboard reviews.

## HubSpot CRM API for Custom Integrations

For agencies with developer resources, the HubSpot API enables deeper integrations. The contacts and deals API endpoints handle most use cases:

```javascript
// Fetch deals for a specific pipeline
const hubspotKey = process.env.HUBSPOT_API_KEY;

async function getPipelineDeals(pipelineId) {
  const response = await fetch(
    `https://api.hubapi.com/crm/v3/objects/deals?properties=dealname,amount,dealstage,pipeline,closedate&pipeline=${pipelineId}&limit=100`,
    {
      headers: {
        'Authorization': `Bearer ${hubspotKey}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  const data = await response.json();
  return data.results;
}
```

This allows building custom dashboards in your own tools, syncing with project management software, or triggering external processes when deals reach specific stages.

## Practical Implementation Order

Setting up HubSpot properly takes time. Prioritize in this order:

1. **Contact properties**: Get basic info capture working first
2. **Deal pipeline**: Match your actual sales stages
3. **Form integration**: Start capturing leads
4. **Basic workflows**: Automate follow-up tasks
5. **Reporting**: Measure what's happening
6. **API integration**: Build custom workflows once foundations exist

This sequence ensures you're collecting useful data before attempting complex automation. Remote agencies benefit particularly from the automation layer—time zone differences make spontaneous follow-ups difficult, so scheduled tasks and notifications fill that gap effectively.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
