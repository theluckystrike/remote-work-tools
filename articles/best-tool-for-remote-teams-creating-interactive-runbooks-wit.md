---
layout: default
title: "Migration runbook example structure"
description: "A practical guide to interactive runbooks with embedded terminal commands for distributed development teams"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Remote Work Tools Guide"
permalink: /best-tool-for-remote-teams-creating-interactive-runbooks-wit/
reviewed: true
score: 9
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools, best-of, remote-work]
---

{% raw %}
For remote teams, use markdown-based runbooks with embedded copy-to-clipboard commands stored in Git—this is the best balance of security, version control, and usability. Tools like Runwayml or custom scripts can display these in an UI with step-by-step validation; the copy-to-terminal model keeps commands out of unauthorized environments while still providing one-click access. Store runbooks in the same repo as infrastructure code so they stay synchronized, and embed variable substitution placeholders (e.g., `$ENVIRONMENT`) that team members fill in before executing commands.

## What Makes Runbooks Interactive

Traditional runbooks read like documentation—they explain what to do but require manual execution. Interactive runbooks embed executable commands directly into the workflow, allowing team members to run them with a single click or copy action that preserves context.

The key components include:

- **Embedded command blocks** that team members can copy or execute directly
- **Environment-specific variables** that adapt commands to different contexts
- **Step-by-step validation** to confirm each action completed successfully
- **Conditional branching** based on outcomes at each stage

## Choosing the Right Tool for Your Team

When evaluating tools for creating interactive runbooks, consider these factors:

**1. Command Execution Model**

Some tools execute commands directly in the browser through a built-in terminal. Others generate commands that users copy to their local terminal. The browser-execution model offers convenience but introduces security considerations. The copy-to-terminal approach maintains separation but requires more user interaction.

**2. Variable and Secret Management**

Effective runbooks need parameterized commands. Look for tools that support variable substitution without exposing secrets in plain text. Environment variables, integration with secret managers, and scoped credentials all matter for production use.

**3. Collaboration Features**

Remote teams need visibility into who created and modified runbooks, version history, and the ability to comment or request changes. Markdown-based runbooks with Git integration provide natural version control.

**4. Output Handling**

Runbooks should capture command output and make it available for debugging. Tools that display terminal output inline help team members verify each step before proceeding.

## Practical Example: Database Migration Runbook

Here's how an interactive runbook might look for a database migration scenario:

```yaml
# Migration runbook example structure
runbook:
  name: production-database-migration
  version: "1.2.0"
  prerequisites:
    - Backup verified
    - Team notified
    - Rollback plan confirmed

steps:
  - name: Verify current connection
    command: |
      pg_isready -h $DB_HOST -p $DB_PORT
    expected_output: "accepting connections"

  - name: Create backup snapshot
    command: |
      pg_dump -Fc -h $DB_HOST -U $DB_USER $DB_NAME > backup_$(date +%Y%m%d_%H%M%S).dump
    timeout: 300

  - name: Run migration scripts
    command: |
      psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f migration_001.sql
    expected_output: "DROP SCHEMA"
```

This structure separates the runbook metadata from executable commands, making it easy to version control and review changes.

## Terminal Integration Patterns

Different tools offer various levels of terminal integration:

**Web-Based Terminals**

Tools like Teleport, JumpServer, and some DevOps platforms embed a terminal directly in the browser. Users authenticate once and can execute commands without local tool configuration.

```bash
# Example: Using a web terminal API
curl -X POST https://runbook.example.com/api/execute \
  -H "Authorization: Bearer $RUNBOOK_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "runbook_id": "db-migration-001",
    "environment": "production",
    "variables": {
      "DB_HOST": "db.prod.internal"
    }
  }'
```

**Local Terminal Execution**

Many teams prefer generating commands for local execution. This approach preserves the user's terminal environment, aliases, and tooling:

```bash
#!/bin/bash
# Generated runbook script
set -e

DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"

echo "=== Verifying database connectivity ==="
pg_isready -h "$DB_HOST" -p "$DB_PORT"

echo "=== Starting migration ==="
psql -h "$DB_HOST" -U postgres -d appdb -f migrate_001.sql

echo "=== Verifying migration ==="
psql -h "$DB_HOST" -U postgres -d appdb -c "SELECT version FROM schema_migrations;"
```

**Hybrid Approaches**

Some platforms combine both—generating local commands while providing output capture and audit logging through a central service. This balances security with convenience.

## Security Considerations

When embedding terminal commands in runbooks, security cannot be an afterthought:

1. **Never embed credentials** in runbook source files. Use environment variables or secret manager integration instead.

2. **Implement command allowlisting** for tools that execute commands directly. Restrict execution to known-safe commands and paths.

3. **Audit trail logging** should capture who executed which runbook, when, and what output resulted. This matters for compliance and incident investigation.

4. **Timeouts and circuit breakers** prevent runaway commands from causing extended outages during incident response.

5. **Access control** ensures only authorized team members can execute production-related runbooks.

## Building a Runbook Library

Start with high-impact, frequently-used procedures:

- **Incident response playbooks** for common failure scenarios
- **Deployment procedures** that span multiple systems
- **Onboarding checklists** that set up new team member environments
- **Maintenance windows** with clear communication templates
- **Rollback procedures** that teams can execute under pressure

Version control your runbooks alongside your code. This practice enables code review for operational procedures and maintains a history of how processes evolved.

## Measuring Runbook Effectiveness

Track these metrics to improve your runbook practice:

- Execution frequency: Which runbooks get used most?
- Time to completion: Do runbooks reduce time-to-resolution?
- Failure rate: Do users encounter errors when following guides?
- Feedback loops: Can users suggest improvements easily?

Regular review sessions where team members walk through runbooks together catch outdated steps and identify gaps.

## Runbook Platform Comparison: Tools for Interactive Execution

Several platforms offer different approaches to making runbooks interactive. Each balances convenience, security, and team needs differently.

### Runwayml ($0-$50/month depending on usage)

Runwayml specializes in API-driven runbook execution with cloud-based terminal sessions. Teams define runbooks in YAML and Runwayml handles the execution environment:

```yaml
# Runwayml runbook configuration
apiVersion: runway.dev/v1
kind: Runbook
metadata:
  name: database-migration-prod
  description: Production database schema migration with rollback capability
spec:
  requiredApprovals:
    - role: database-admin
    - role: platform-lead

  variables:
    - name: ENVIRONMENT
      type: string
      default: production
      allowedValues: [staging, production]
    - name: MIGRATION_VERSION
      type: string
      pattern: '^\d+\.\d+\.\d+$'
      description: Semantic version of migration script

  steps:
    - name: Pre-flight checks
      timeout: 5m
      command: |
        #!/bin/bash
        set -e
        echo "Verifying database connectivity..."
        pg_isready -h $DB_HOST -p $DB_PORT
        echo "Checking migration script exists..."
        ls -la migrations/v${MIGRATION_VERSION}/
        echo "Pre-flight checks passed"

    - name: Create backup
      timeout: 30m
      command: |
        #!/bin/bash
        set -e
        BACKUP_NAME="backup_$(date +%Y%m%d_%H%M%S).dump"
        pg_dump -Fc -h $DB_HOST -U $DB_USER $DB_NAME > /backups/${BACKUP_NAME}
        echo "Backup created: ${BACKUP_NAME}"

    - name: Run migration
      timeout: 15m
      command: |
        #!/bin/bash
        set -e
        psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f migrations/v${MIGRATION_VERSION}/up.sql
        echo "Migration completed"

    - name: Verify migration
      timeout: 5m
      command: |
        #!/bin/bash
        psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT * FROM schema_migrations ORDER BY version DESC LIMIT 5;"

  rollback:
    command: |
      #!/bin/bash
      psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f migrations/v${MIGRATION_VERSION}/down.sql
      echo "Rollback completed"
```

Runwayml provides a web interface where authorized users can fill in variables, review the command sequence, and execute with audit logging. Output streams in real-time to the browser with color-coded success/failure markers.

**Strengths**: Built-in approval workflows, variable validation, timeout enforcement, audit logging stored for 90 days.

**Limitations**: Requires Runwayml-specific YAML syntax; limited to their execution infrastructure; pricing scales with execution minutes.

### Teleport ($0 enterprise, self-hosted)

Teleport offers zero-trust access to servers and Kubernetes clusters with embedded runbook capability. Teams use Teleport to provision short-lived credentials and execute commands through centralized infrastructure:

```bash
# Define runbook in Teleport using tctl
cat > /tmp/runbook.yaml <<EOF
kind: RuntimeScript
version: v1
metadata:
  name: deployment-workflow
spec:
  script: |
    #!/bin/bash
    set -e
    # Deploy to Kubernetes cluster
    kubectl set image deployment/app app=myapp:$VERSION --namespace=$NAMESPACE
    kubectl rollout status deployment/app --namespace=$NAMESPACE

  parameters:
    - name: VERSION
      description: Docker image tag to deploy
      type: string
    - name: NAMESPACE
      description: Kubernetes namespace
      type: string
      default: production
EOF

tctl create -f /tmp/runbook.yaml

# Execute runbook with audit logging
tctl exec -f /tmp/runbook.yaml -p VERSION=v2.3.1 -p NAMESPACE=production
```

Teleport maintains an audit log of every command execution with user identity, timestamp, and full session recording. Teams can replay sessions for incident investigation or compliance audits.

**Strengths**: Zero-trust architecture, session recording with playback, works with any SSH-accessible server, self-hosted option eliminates cloud dependency.

**Limitations**: Steeper learning curve; requires infrastructure changes (agent deployment on servers); free tier limited to single cluster.

### GitHub Actions (Free-$21/month for enterprise runners)

For teams already using GitHub, Actions provides a natural place to embed runbooks. Create public documentation with embedded "Run Workflow" buttons that trigger predefined Actions:

```yaml
# .github/workflows/incident-response.yml
name: Incident Response Runbook
on:
  workflow_dispatch:
    inputs:
      incident_id:
        description: 'Incident tracking ID'
        required: true
      severity:
        description: 'Severity level'
        required: true
        type: choice
        options:
          - low
          - medium
          - high
          - critical

jobs:
  respond:
    runs-on: ubuntu-latest
    steps:
      - name: Acknowledge incident
        run: |
          echo "Incident ID: ${{ github.event.inputs.incident_id }}"
          echo "Severity: ${{ github.event.inputs.severity }}"
          # Notify incident management system
          curl -X POST https://incidents.example.com/api/acknowledge \
            -H "Authorization: Bearer ${{ secrets.INCIDENT_API_KEY }}" \
            -d '{
              "incident_id": "${{ github.event.inputs.incident_id }}",
              "acknowledged_by": "${{ github.actor }}",
              "timestamp": "'$(date -Iseconds)'"
            }'

      - name: Disable traffic to affected service
        if: github.event.inputs.severity == 'critical'
        run: |
          aws elb deregister-instances-from-load-balancer \
            --load-balancer-name prod-lb \
            --instances i-0123456789abcdef0 \
            --region us-east-1

      - name: Create incident timeline entry
        run: |
          # Append to incident log
          echo "$(date -Iseconds) - ${{ github.actor }} executed incident response" >> incidents/${{ github.event.inputs.incident_id }}.log
```

In your runbook documentation, embed a button users click to execute:

```markdown
## Emergency: Production Database Under Load

If the production database exceeds 85% CPU for 5+ minutes:

1. **Acknowledge the incident** in our incident system
2. **Reduce load** by temporarily routing read traffic to replicas
3. **Monitor recovery** in the dashboard

[Run Incident Response Workflow](https://github.com/yourorg/infrastructure/actions/workflows/incident-response.yml)

After triggering, provide:
- Incident ID (from PagerDuty alert)
- Severity level (critical if customer-facing)
```

**Strengths**: Free for public repos; integrates with GitHub-based workflows; natural audit trail through GitHub Actions logs.

**Limitations**: Execution limited to GitHub infrastructure; less suitable for real-time interactive terminals.

### Self-Hosted Approach: Markdown + Script Generation

For maximum control and minimal external dependencies:

```bash
#!/bin/bash
# runbook_cli.sh - Generate executable scripts from markdown runbooks

# Parse runbook markdown, extract code blocks, generate shell script
generate_runbook_script() {
  local runbook_file=$1
  local environment=$2

  # Extract code blocks marked with ```bash
 awk '
 /^```bash/{flag=1; next}
    /^```/{flag=0; next}
 flag {print}
 ' "$runbook_file" | \
 # Substitute environment variables
 sed "s|\$ENVIRONMENT|$environment|g" | \
 sed "s|\$TIMESTAMP|$(date +%Y%m%d_%H%M%S)|g" \
 > "/tmp/runbook_generated_$environment.sh"

 chmod +x "/tmp/runbook_generated_$environment.sh"
 echo "Generated runbook script: /tmp/runbook_generated_$environment.sh"
}

# Execute with confirmation prompts
execute_with_confirmation() {
 local script=$1

 echo "=== RUNBOOK EXECUTION ==="
 echo "Script: $script"
 echo ""
 echo "Commands to execute:"
 echo "---"
 cat "$script"
 echo "---"
 echo ""
 read -p "Proceed with execution? (type 'yes' to confirm): " confirmation

 if [ "$confirmation" = "yes" ]; then
 bash "$script"
 else
 echo "Execution cancelled"
 fi
}

# Usage
generate_runbook_script "runbooks/database-migration.md" "production"
execute_with_confirmation "/tmp/runbook_generated_production.sh"
```

Store runbooks as markdown in Git alongside your infrastructure code:

```markdown
# Database Migration Runbook v1.2.0

**Prerequisites**:
- [ ] Backup verified
- [ ] Team notified
- [ ] Rollback plan confirmed

## Steps

### 1. Verify Connection

```bash
pg_isready -h $DB_HOST -p $DB_PORT
psql -h $DB_HOST -U postgres -d postgres -c "SELECT version();"
```

Expected output: Should return PostgreSQL version without connection errors.

### 2. Create Backup

```bash
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).dump"
pg_dump -Fc -h $DB_HOST -U $DB_USER $DB_NAME > /secure/backups/$BACKUP_FILE
echo "Backup complete: $BACKUP_FILE"
```

This command may take 5-30 minutes depending on database size.

### 3. Run Migration

```bash
psql -h $DB_HOST -U $DB_USER -d $DB_NAME < migrations/001_add_users_table.sql
```

Success: Table created, no errors in output.

### 4. Verify Schema Changes

```bash
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "\\d users"
```

Check that new columns match migration specification.

## Rollback (if needed)

```bash
psql -h $DB_HOST -U $DB_USER -d $DB_NAME < migrations/001_rollback.sql
```

After rollback, verify original schema with step 4.
```

This approach keeps runbooks version-controlled, auditable via Git history, and executable without external platform dependencies.

## Runbook Template Library for Common Operations

### Deployment Runbook

```yaml
Title: Blue-Green Deployment
Purpose: Deploy new version with zero downtime
Estimated Duration: 15 minutes
Rollback Duration: 5 minutes

Steps:
 1. Health Check Green (Standby) Environment
 2. Deploy Application to Green
 3. Run Smoke Tests Against Green
 4. Update Load Balancer to Route 10% to Green
 5. Monitor Error Rates (5 minutes)
 6. Route 50% to Green
 7. Monitor Error Rates (5 minutes)
 8. Route 100% to Green
 9. Decommission Blue Environment

Rollback Trigger: Error rate > 0.1% or response time > 2s
Rollback Action: Immediately route 100% back to Blue
```

### Incident Response Runbook

```yaml
Title: Production API Outage Response
Purpose: Rapid diagnosis and remediation
Estimated Duration: 10 minutes to mitigation

Steps:
 1. Page on-call engineer
 2. Check API health dashboard
 3. Review application logs (last 5 minutes)
 4. Check infrastructure metrics (CPU, memory, network)
 5. Determine if issue is:
 a. Code-related → Rollback last deployment
 b. Infrastructure → Scale up resources
 c. Dependency → Failover to secondary
 6. Communicate status to stakeholders
 7. Create incident timeline
 8. Schedule post-mortem

Escalation: If unresolved after 5 minutes, page incident commander
```

### Onboarding Runbook

```yaml
Title: New Developer Environment Setup
Purpose: Bootstrap development environment
Estimated Duration: 45 minutes

Steps:
 1. Clone repository with SSH keys
 2. Install dependencies (npm/pip/etc)
 3. Configure database connection
 4. Run database migrations
 5. Start development server
 6. Verify health check endpoint
 7. Run test suite
 8. Create personal feature branch

Validation: Developer can run tests locally without errors
```

## Integrating Runbooks into Incident Response Workflow

Link runbooks directly into your incident management system:

```yaml
PagerDuty Integration:
 - Create escalation policy with runbook link in description
 - Include runbook URL in incident alert notification
 - Teams acknowledge alert and click runbook link
 - Runbook execution triggers audit log entry back to incident
 - Post-incident review includes runbook effectiveness assessment
```

This closes the loop between detection, response, and continuous improvement.


## Related Articles

- [From your local machine with VPN active](/remote-work-tools/remote-team-runbook-creation-guide-for-incident-response-wit/)
- [Example: project-update.yml - Scheduled updates structure](/remote-work-tools/how-to-manage-client-expectations-when-team-works-asynchrono/)
- [Example: Benefit request data structure](/remote-work-tools/return-to-office-childcare-benefit-policy-template-for-hybri/)
- [Remote Team Runbook Template for Database Failover](/remote-work-tools/remote-team-runbook-template-for-database-failover-procedure/)
- [Remote Team Runbook Template for Deploying Hotfix to](/remote-work-tools/remote-team-runbook-template-for-deploying-hotfix-to-product/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
