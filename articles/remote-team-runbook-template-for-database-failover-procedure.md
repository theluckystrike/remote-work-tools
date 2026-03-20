---
layout: default
title: "Remote Team Runbook Template for Database Failover"
description: "A practical runbook template for database failover procedures designed for remote DevOps teams working across multiple time zones with async."
date: 2026-03-16
author: theluckystrike
permalink: /remote-team-runbook-template-for-database-failover-procedure/
categories: [guides]
tags: [database, failover, devops, remote-work, runbook, distributed-teams, incident-response]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# Remote Team Runbook Template for Database Failover Procedure with Distributed DevOps Staff

When your primary database instance fails at 3 AM while your DBA is eight time zones away, the difference between a 15-minute recovery and a multi-hour outage often comes down to having a well-practiced failover runbook. Database failures don't wait for business hours, and distributed DevOps teams can't rely on synchronous handoffs during critical incidents. This guide provides a runbook template that remote engineering teams can adapt for handling database failovers across distributed staff.

## The Challenge of Database Failover in Distributed Teams

Traditional database operations assume that the person with the most knowledge about the system is available when problems arise. In distributed teams spanning multiple time zones, this assumption breaks down. A failover that requires senior DBA approval can stall for hours simply because the right person is sleeping.

Effective remote team runbooks solve this by establishing clear decision criteria, predefined automation, and async-compatible verification steps. The goal is not to eliminate human judgment but to structure decisions so that qualified team members can make informed choices without waiting for specific individuals.

## Pre-Failover Preparation Checklist

Before any failover scenario, ensure these prerequisites are in place:

1. **Replication topology is documented** — Know your primary-replica configuration, replication lag thresholds, and any read-replica hierarchies
2. **Failover credentials are configured** — All authorized engineers have appropriate access to execute failover commands
3. **Monitoring alerts are tuned** — Set clear thresholds for when to trigger failover (not every blip warrants it)
4. **Communication channels are established** — Know where to post status updates and who to notify

### Database Connection String Template

Store your connection information in a standardized format accessible to all team members:

```yaml
# config/database-failover.yaml
production:
  primary:
    host: db-primary.internal
    port: 5432
    region: us-east-1
  replicas:
    - host: db-replica-1.internal
      region: us-east-1
      priority: 1
    - host: db-replica-2.internal
      region: us-west-2
      priority: 2
  failover:
    trigger_conditions:
      cpu_percent: 90
      connection_count: 1000
      replication_lag_seconds: 30
      error_rate_percent: 5
    auto_failover_enabled: true
    max_replication_lag_before_failover: 30
```

## The Database Failover Runbook

### Phase 1: Detection and Initial Assessment (0-3 minutes)

**Trigger:** Monitoring alert or on-call engineer notification

**Actions:**

```
1. Acknowledge the alert in #database-alerts Slack channel
2. Verify the alert is not a false positive (check metrics directly)
3. Identify current database topology status
4. Determine if this is a primary or replica failure
5. Check replication lag across all replicas
```

**Automated Detection Script:**

```bash
#!/bin/bash
# db-health-check.sh - Run this on detection

DB_PRIMARY="${DB_PRIMARY_HOST:-db-primary.internal}"
DB_REPLICAS=("db-replica-1.internal" "db-replica-2.internal")
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

echo "=== Database Health Check ==="
echo "Primary: $DB_PRIMARY"

# Check primary availability
if pg_isready -h "$DB_PRIMARY" -p 5432 -U readonly; then
    echo "✅ Primary is reachable"
else
    echo "❌ Primary is NOT reachable"
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      --data '{"text": "🚨 DATABASE ALERT: Primary database unreachable!"}'
fi

# Check replica replication lag
for replica in "${DB_REPLICAS[@]}"; do
    LAG=$(psql -h "$replica" -U readonly -t -c "SELECT now() - pg_last_xact_replay_timestamp();" 2>/dev/null | xargs)
    echo "Replica $replica lag: $LAG"
done
```

### Phase 2: Decision to Failover (3-10 minutes)

Use this decision matrix to determine if failover is appropriate:

| Condition | Action |
|-----------|--------|
| Primary unreachable, replicas healthy | Initiate failover to most current replica |
| Primary reachable but degraded | Investigate root cause first; failover if no improvement in 10 minutes |
| Replica lag > 30 seconds | Do not failover; investigate replication issues |
| Multiple replicas down | Assess scope; consider data loss scenarios |

**Async Decision Protocol for Distributed Teams:**

When the on-call engineer is not the primary DBA:

1. Post the current situation in #database-incidents with the `?failover` tag
2. Include: current metrics, time since failure, affected services
3. Wait 3 minutes for objections or additional context
4. If no blocking objections, proceed with failover
5. Document all decisions in the incident thread

```markdown
## Failover Decision Request

**Current Status:** [Primary unreachable / High latency / etc]
**Time Since Issue:** [X minutes]
**Affected Services:** [list]
**Replication Status:** [replica-1: Xs lag, replica-2: Ys lag]

**Recommendation:** Failover to [replica name]

@oncall-dba @senior-engineer - Any objections to proceeding?
```

### Phase 3: Failover Execution (10-20 minutes)

Execute the failover using your database's native tools. The following examples use PostgreSQL with Patroni, but adapt to your specific setup:

**Step 1: Promote the Target Replica**

```bash
#!/bin/bash
# promote-replica.sh - Execute on the replica to promote

REPLICA_HOST="${1:-db-replica-1.internal}"
NEW_PRIMARY="$REPLICA_HOST"

echo "Promoting $REPLICA_HOST to primary..."

# For Patroni-managed clusters
patronictl -c /etc/patroni.yml promote "$REPLICA_HOST"

# For manual PostgreSQL
# pg_ctl promote -D /var/lib/postgresql/data

# Verify promotion succeeded
if pg_isready -h "$NEW_PRIMARY" -p 5432; then
    echo "✅ Failover completed successfully"
    
    # Notify the team
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      --data '{"text": "✅ Database failover completed. New primary: '"$NEW_PRIMARY"'"}'
else
    echo "❌ Failover verification failed"
    exit 1
fi
```

**Step 2: Update Application Connection Strings**

```bash
#!/bin/bash
# update-db-dns.sh - Point applications to new primary

NEW_PRIMARY_IP=$(host "$NEW_PRIMARY" | awk '{print $NF}')

# For service discovery (Consul example)
consul kv put database/primary/host "$NEW_PRIMARY_IP"

# For environment-based deployments
aws ssm put-parameter \
  --name "/app/production/db_host" \
  --value "$NEW_PRIMARY_IP" \
  --type "String" \
  --overwrite

# Restart application pods to pick up new connections
kubectl rollout restart deployment/production-api
```

**Step 3: Verify Replication to Old Primary (Now Replica)**

```bash
#!/bin/bash
# verify-new-replica.sh - Run on the old primary

OLD_PRIMARY="db-primary.internal"
NEW_PRIMARY="db-replica-1.internal"

# Configure old primary as replica of new primary
# This depends on your specific replication setup
psql -h "$OLD_PRIMARY" -c "SELECT pg_start_replay();"

# Monitor until caught up
while true; do
    LAG=$(psql -h "$OLD_PRIMARY" -t -c "SELECT now() - pg_last_xact_replay_timestamp();")
    echo "Replication lag: $LAG"
    if [ "$LAG" = "00:00:00" ]; then
        echo "✅ Fully caught up"
        break
    fi
    sleep 5
done
```

### Phase 4: Post-Failover Verification (20-30 minutes)

Run checks to ensure the failover was successful:

```bash
#!/bin/bash
# post-failover-verification.sh

NEW_PRIMARY="${1:-db-replica-1.internal}"

echo "=== Post-Failover Verification ==="

# 1. Basic connectivity
echo "1. Testing connectivity..."
pg_isready -h "$NEW_PRIMARY" || exit 1

# 2. Query execution
echo "2. Testing query execution..."
psql -h "$NEW_PRIMARY" -c "SELECT 1;" | grep -q "1" || exit 1

# 3. Application smoke test
echo "3. Running application smoke tests..."
SMOKE=$(curl -s https://api.yourapp.com/health/db)
echo "$SMOKE" | grep -q "healthy" || exit 1

# 4. Test write operations
echo "4. Testing write operations..."
WRITE_TEST=$(psql -h "$NEW_PRIMARY" -c "CREATE TABLE IF NOT EXISTS failover_test (id SERIAL, ts TIMESTAMP DEFAULT NOW()); DROP TABLE failover_test;" 2>&1)
echo "$WRITE_TEST" | grep -q "DROP" || exit 1

# 5. Verify backup jobs
echo "5. Checking backup jobs..."
pg_dump -h "$NEW_PRIMARY" -F c -f /tmp/backup_test.sql

echo "✅ All verification checks passed"
```

### Phase 5: Incident Documentation and Follow-Up

Create an incident report within 24 hours of the failover:

```markdown
## Database Failover Incident Report

**Date:** [ISO timestamp]
**Duration:** [start to full resolution]
**Root Cause:** [what caused the original failure]

### Timeline
- 02:15 - Alert triggered: primary unreachable
- 02:18 - Failover decision made (async approval)
- 02:25 - Failover completed
- 02:35 - All systems verified

### What Went Well
- [List successful aspects]

### What Needs Improvement
- [List action items]

### Action Items
- [ ] Schedule post-mortem review
- [ ] Update runbook with lessons learned
- [ ] Review monitoring thresholds
```

## Key Principles for Remote Team Database Failovers

**Automate detection but humanize decisions.** Automated monitoring should catch issues immediately, but failover execution benefits from human oversight when time permits. Structure your runbook so that clear-cut cases can proceed automatically while ambiguous situations trigger async escalation.

**Document your topology in code.** Keep your database configuration in version control alongside your application code. This ensures every team member can access current infrastructure information regardless of time zone.

**Practice the runbook regularly.** Schedule quarterly failover drills. Test the process with a non-production database to identify gaps before real incidents expose them.

**Establish clear ownership rotation.** Ensure that failover authority is not limited to a single person. Train multiple team members and rotate on-call schedules to provide coverage across time zones.

## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [Remote Team Runbook Template for Deploying Hotfix to.](/remote-work-tools/remote-team-runbook-template-for-deploying-hotfix-to-product/)
- [Remote Team Security Incident Response Plan Template for.](/remote-work-tools/remote-team-security-incident-response-plan-template-for-distributed-organizations-guide/)
- [Remote Team Runbook Template for SSL Certificate Renewal.](/remote-work-tools/remote-team-runbook-template-for-ssl-certificate-renewal-pro/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
