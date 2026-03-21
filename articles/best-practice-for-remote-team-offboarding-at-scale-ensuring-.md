---
layout: default
title: "Find all GitHub repositories where user is admin"
description: "A practical guide for engineering managers and developers handling remote team offboarding at scale, with actionable scripts and workflows"
date: 2026-03-16
author: theluckystrike
permalink: /best-practice-for-remote-team-offboarding-at-scale-ensuring-/
categories: [guides]
tags: [remote-work-tools, tools, best-of, remote-work]
reviewed: true
score: 8
voice-checked: true
intent-checked: true
---

{% raw %}
Scale remote offboarding by running two parallel tracks: knowledge transfer and access removal, executed through automated scripts and structured handoff documents rather than informal conversations. Without physical handshakes or office walkthroughs, departing employees retain dangerous system access while critical knowledge walks out the door. This framework provides actionable checklists and automation code for systematic offboarding across distributed teams.

## The Two-Phase Offboarding Framework

Effective remote offboarding divides into two parallel tracks: **knowledge transfer** and **access removal**. Running these tracks simultaneously prevents the common failure mode where departing employees retain system access while critical knowledge walks out the door.

### Phase 1: Knowledge Transfer Architecture

Knowledge transfer in remote teams requires explicit documentation because informal hallway conversations simply do not exist. Build a structured handoff package that includes:

**Repository Ownership Transfer**
Every code repository, configuration file, and deployment pipeline the departing engineer owned needs explicit reassignment. Create a script that identifies all resources where they are the primary owner:

```bash
#!/bin/bash
# Find all GitHub repositories where user is admin
gh repo list --limit 1000 --json name,owner,permissions \
  | jq '.[] | select(.permissions.admin == true) | .full_name' \
  | while read repo; do
      echo "Review ownership transfer for: $repo"
    done
```

**Runbook and Documentation Review**
Archive any runbooks, architecture decision records, or internal wikis the departing team member authored. Use a simple grep across your documentation repository:

```bash
grep -r "author: $DEPARTING_EMAIL" docs/ --include="*.md" | cut -d: -f1 | uniq
```

**Communication Context**
Export relevant Slack or Discord channels where the departing engineer provided technical context. Many teams use automated exports through the platform APIs:

```python
import os
from slack_sdk import WebClient

def export_engineer_contributions(user_id, channel_ids):
    client = WebClient(token=os.environ['SLACK_TOKEN'])
    contributions = []

    for channel in channel_ids:
        result = client.conversations_history(
            channel=channel,
            limit=1000
        )
        contributions.extend([
            msg for msg in result['messages']
            if msg.get('user') == user_id
        ])

    return contributions
```

### Phase 2: Access Removal Automation

Access revocation must happen in order of sensitivity. Start with the most critical systems and work outward. A typical sequence includes:

1. **Identity and SSO systems** (Okta, Auth0, Google Workspace)
2. **Cloud infrastructure** (AWS, GCP, Azure)
3. **Code repositories** (GitHub, GitLab, Bitbucket)
4. **CI/CD pipelines** (GitHub Actions, CircleCI, Jenkins)
5. **Communication platforms** (Slack, Teams, Discord)
6. **Project management** (Jira, Linear, Asana)

#### Terraform-Based Access Revocation

If your organization uses infrastructure-as-code, use it for access management:

```hcl
# Module for offboarding user access
variable "departing_user_email" {
  type = string
}

resource "aws_iam_user" "departing_user" {
  name = var.departing_user_email
  # This will be destroyed, removing all attached policies
}

# Run after knowledge transfer complete
# terraform destroy -var-file=offboarding.tfvars
```

For GitHub organization management, a dedicated offboarding action automates the heavy lifting:

```yaml
name: Offboard Engineer
on:
  workflow_dispatch:
    inputs:
      username:
        required: true

jobs:
  remove-access:
    runs-on: ubuntu-latest
    steps:
      - name: Remove from organization
        run: |
          gh org remove-member ${{ github.repository_owner }} ${{ github.event.inputs.username }}

      - name: Revoke personal access tokens
        run: |
          gh api -X DELETE /users/${{ github.event.inputs.username }}/tokens

      - name: Archive their repositories
        run: |
          # Script to transfer ownership to team accounts
```

## Scaling Offboarding Across Multiple Departures

When executing coordinated reductions or rapid scaling down, manual offboarding becomes a bottleneck. Build an offboarding queue that processes departures in parallel while maintaining audit trails.

### The Offboarding Checklist Template

Create a standardized checklist that works for any role:

```markdown
## Departing Employee: {{name}}
## Last Day: {{date}}
## Replacement: {{replacement}}

### Knowledge Transfer (Complete by Last Day - 3)
- [ ] Code ownership transferred for all repositories
- [ ] Documentation authorship updated
- [ ] Runbook review meeting completed
- [ ] Customer-facing Escalation contacts updated
- [ ] Internal knowledge base articles reviewed

### Access Revocation (Complete by Last Day)
- [ ] SSO/Identity provider account disabled
- [ ] Cloud console access removed
- [ ] Database access revoked
- [ ] VPN credentials revoked
- [ ] Email distribution lists updated
- [ ] Calendar sharing permissions removed
- [ ] Slack/Teams account deactivated

### Hardware and Assets (Complete by Last Day + 2)
- [ ] Company laptop returned via tracked shipping
- [ ] Hardware keys (YubiKey, etc.) returned
- [ ] 2FA recovery codes transferred to IT
```

### Automating the Checklist

Connect your checklist to your HRIS system to auto-populate departure dates and trigger automation:

```python
from datetime import datetime, timedelta

def process_offboarding_queue(hris_client, it_client):
    """Process departures scheduled within 7 days"""
    departing = hris_client.get_departures(
        date_range=[
            datetime.now(),
            datetime.now() + timedelta(days=7)
        ]
    )

    for employee in departing:
        # Trigger access revocation workflow
        it_client.trigger_offboarding(
            user_id=employee.id,
            scheduled_date=employee.last_day,
            parallel_tasks=[
                "revoke_cloud_access",
                "revoke_sso",
                "export_slack_messages",
                "transfer_repository_ownership"
            ]
        )
```

## Common Pitfalls to Avoid

The "I" Problem: Avoid offloading all offboarding work onto the departing employee. They may not feel motivated to document everything thoroughly. Instead, assign a peer to review and supplement their handoff.

Temporal Gaps: Run knowledge transfer sessions at least one week before departure. Rushing this process guarantees gaps in institutional knowledge.

Access Blind Spots: Remote teams often accumulate shadow IT—personal API keys, temporary deployment accounts, or individual SaaS subscriptions. Query your cloud billing and audit logs to catch these:

```bash
# Find AWS access keys created in last 90 days
aws iam list-access-keys --region us-east-1 \
  | jq '.AccessKeyMetadata[] | select(.CreateDate > (now - 7776000))'
```

Incomplete Communication Updates: Missing email updates cause support escalations to bounce to inactive accounts. Verify all aliases and forwarding rules before deactivating accounts.

## Measuring Offboarding Effectiveness

Track these metrics to improve your process over time:

- **Mean time to complete access revocation** (target: under 24 hours post-departure)
- **Knowledge base gaps reported by successors** (target: zero critical gaps)
- **Unclaimed resource cleanup** (track orphaned cloud resources monthly)

Build a simple dashboard:

```sql
SELECT
  DATE(last_day) as departure_date,
  AVG(TIMESTAMPDIFF(HOUR, last_day, access_removed_at)) as hours_to_revocation,
  COUNT(*) as departures
FROM offboarding_log
GROUP BY DATE(last_day)
ORDER BY departure_date DESC;
```


## Related Articles

- [Best Practice for Remote Team README Files in Repositories](/remote-work-tools/best-practice-for-remote-team-readme-files-in-repositories-s/)
- [Best Practice for Remote Team Escalation Paths That Scale](/remote-work-tools/best-practice-for-remote-team-escalation-paths-that-scale-wi/)
- [Find the first commit by a specific author](/remote-work-tools/best-practice-for-measuring-remote-onboarding-effectiveness-with-time-to-first-commit/)
- [Example: Junior Engineer Competency Matrix](/remote-work-tools/remote-team-interviewer-calibration-process-for-ensuring-con/)
- [Example: Find pages not modified in the last 180 days using](/remote-work-tools/how-to-create-remote-team-documentation-sprint-dedicating-ti/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
