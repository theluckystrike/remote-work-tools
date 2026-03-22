---
layout: default
title: "How to Create Remote Team Runbook Templates"
description: "Build reusable runbook templates for remote engineering teams covering incident response, deployments, and database procedures with checklist automation"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-remote-team-runbook-templates/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Runbooks turn undocumented institutional knowledge into step-by-step procedures anyone on the team can follow at 3am. Good runbooks are opinionated, tested, and short — they list commands to run, not theory to understand. This guide builds the templates and tooling for a remote engineering team's runbook library.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Prerequisites](#prerequisites)
- [Prerequisites](#prerequisites)
- [Prerequisites](#prerequisites)
- [Prerequisites](#prerequisites)
- [Troubleshooting](#troubleshooting)
- [Related Reading](#related-reading)

# [Operation Name] Runbook

**Owner:** @team-name
**Last tested:** YYYY-MM-DD
**Estimated time:** N minutes
**Severity:** Critical / High / Medium / Low

### Step 2: Purpose
One sentence: what does this runbook do?

## Prerequisites
- Access required
- Tools needed
- Checks to do first

### Step 3: Steps
1. Step with command
2. Step with expected output
3. Verification step

### Step 4: Verification
How to confirm the operation succeeded.

### Step 5: Rollback
How to undo this if something goes wrong.

### Step 6: Escalation
Who to page if this doesn't work.
```

### Step 7: Template 1: Service Restart

```markdown
# Service Restart Runbook

**Owner:** @platform-team
**Last tested:** 2026-03-15
**Estimated time:** 5 minutes
**Severity:** High

### Step 8: Purpose
Safely restart a production service without extended downtime.

## Prerequisites
- SSH access to production servers
- Confirm: `kubectl get pods -n production` (for k8s) or SSH access
- Alert #incidents that restart is in progress

### Step 9: Steps

### Kubernetes

1. Check current pod status:
   ```bash
   kubectl get pods -n production -l app=your-service
   ```
 Expected: All pods in `Running` state before proceeding.

2. Scale down to zero (optional for critical services):
   ```bash
   kubectl scale deployment your-service -n production --replicas=0
   kubectl wait --for=delete pods -l app=your-service -n production --timeout=60s
   ```

3. Restart with rolling update (preferred):
   ```bash
   kubectl rollout restart deployment/your-service -n production
   ```

4. Monitor rollout:
   ```bash
   kubectl rollout status deployment/your-service -n production --timeout=120s
   ```
 Expected output: `deployment "your-service" successfully rolled out`

### Docker / systemd

1. Check service health before restart:
   ```bash
   systemctl status your-service
   ```

2. Restart:
   ```bash
   sudo systemctl restart your-service
   ```

3. Check for errors:
   ```bash
   sudo journalctl -u your-service -n 50 --no-pager
   ```

### Step 10: Verification

```bash
# Check service responds
curl -s --max-time 10 https://api.example.com/health | jq .
# Expected: {"status": "ok"}

# Check error rate in Grafana:
# Dashboard: Service Health > Error Rate > last 5 minutes
# Expected: < 0.1% errors
```

### Step 11: Rollback

If the service doesn't come back up:

```bash
# Kubernetes: rollback to previous version
kubectl rollout undo deployment/your-service -n production
kubectl rollout status deployment/your-service -n production

# Docker: start previous container
docker start your-service_previous
```

### Step 12: Escalation

Service still down after 10 minutes: page @on-call-engineer via PagerDuty.
```

### Step 13: Template 2: Database Backup Verification

```markdown
# Database Backup Verification Runbook

**Owner:** @database-team
**Last tested:** 2026-03-01
**Estimated time:** 20 minutes
**Severity:** Medium

### Step 14: Purpose
Verify that recent database backup is valid and can be restored.

## Prerequisites
- Access to backup storage (S3/MinIO)
- Test restore environment available
- At least 10GB free disk space on test host

### Step 15: Steps

1. List recent backups and confirm latest is recent:
   ```bash
   mc ls company/backups/postgres/ --recursive | sort | tail -10
   # Confirm latest backup is < 24 hours old
   ```

2. Download latest backup to test host:
   ```bash
   BACKUP=$(mc ls company/backups/postgres/ --recursive | sort | tail -1 | awk '{print $NF}')
   mc cp "company/backups/postgres/${BACKUP}" /tmp/test-restore.sql.gz
   echo "Backup size: $(du -sh /tmp/test-restore.sql.gz)"
   ```

3. Create test database:
   ```bash
   createdb -U postgres test_restore_$(date +%Y%m%d)
   ```

4. Restore backup:
   ```bash
   DB_NAME="test_restore_$(date +%Y%m%d)"
   gunzip -c /tmp/test-restore.sql.gz | psql -U postgres "$DB_NAME"
   ```

5. Verify table count matches production:
   ```bash
   # On production:
   psql -U postgres appdb -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';"

   # On test restore:
   psql -U postgres "$DB_NAME" -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';"
   # Counts should match
   ```

6. Verify recent data exists:
   ```bash
   psql -U postgres "$DB_NAME" \
     -c "SELECT MAX(created_at) FROM orders;"
   # Should be within last 24 hours
   ```

### Step 16: Cleanup

```bash
dropdb -U postgres "test_restore_$(date +%Y%m%d)"
rm /tmp/test-restore.sql.gz
```

### Step 17: Verification

Record backup test results in the backup log:
```
Date: YYYY-MM-DD
Backup file: filename.sql.gz
Backup size: NNN MB
Restore time: N minutes
Table count match: YES/NO
Latest data date: YYYY-MM-DD
Tested by: @username
```

### Step 18: Escalation

Backup older than 36 hours or restore fails: page @database-team immediately.
```

### Step 19: Template 3: SSL Certificate Renewal

```markdown
# SSL Certificate Renewal Runbook

**Owner:** @platform-team
**Last tested:** 2026-01-10
**Estimated time:** 15 minutes (automated) / 45 minutes (manual)

### Step 20: Purpose
Renew SSL certificates before expiry. Run this 30 days before expiry.

## Prerequisites
- Root/sudo access to servers running nginx/apache
- Certbot installed, or access to certificate provider dashboard

### Step 21: Check Current Expiry

```bash
# Check all certs on a server
for domain in api.example.com git.example.com auth.example.com; do
  echo -n "$domain: "
  echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -dates 2>/dev/null | grep notAfter
done
```

### Step 22: Automated Renewal (Let's Encrypt)

```bash
# Test renewal (dry run)
sudo certbot renew --dry-run

# Renew all certs
sudo certbot renew

# Reload nginx after renewal
sudo systemctl reload nginx

# Verify renewal
sudo certbot certificates
```

### Step 23: Manual Renewal (Other CA)

1. Generate new CSR:
   ```bash
   openssl req -new -newkey rsa:2048 -nodes \
     -keyout /etc/ssl/private/example.com.key \
     -out /tmp/example.com.csr \
     -subj "/C=US/ST=NY/O=YourCompany/CN=example.com"
   ```

2. Submit CSR to your CA, download new certificate.

3. Install new certificate:
   ```bash
   sudo cp new-cert.crt /etc/ssl/certs/example.com.crt
   sudo nginx -t && sudo systemctl reload nginx
   ```

### Step 24: Verification

```bash
# Verify new expiry date
echo | openssl s_client -servername api.example.com \
  -connect api.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates
# notAfter should be 90 days from now (Let's Encrypt) or per CA
```
```

### Step 25: Run book CI — Auto-Test Commands

Test runbook commands don't drift from reality:

```yaml
# .github/workflows/test-runbooks.yml
name: Test Runbook Commands

on:
 schedule:
 - cron: '0 6 * * 1' # Weekly Monday
 pull_request:
 paths: ['runbooks/**']

jobs:
 test-cert-check:
 runs-on: ubuntu-latest
 steps:
 - uses: actions/checkout@v4
 - name: Test certificate check command
 run: |
 echo | openssl s_client -servername google.com \
 -connect google.com:443 2>/dev/null \
 | openssl x509 -noout -dates
```

### Step 26: Run book Index Template

```markdown
# Runbook Index

### Step 27: Plan Incident Response
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [Service Restart](./service-restart.md) | @platform | 2026-03-15 | 5m |
| [Database Failover](./db-failover.md) | @dba | 2026-02-01 | 30m |
| [High Traffic Response](./high-traffic.md) | @sre | 2026-03-01 | 15m |

### Step 28: Deploy ments
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [Deploy Hotfix](./deploy-hotfix.md) | @engineering | 2026-03-10 | 20m |
| [Rollback Release](./rollback.md) | @engineering | 2026-03-05 | 10m |

### Step 29: Perform Maintenance
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [SSL Renewal](./ssl-renewal.md) | @platform | 2026-01-10 | 15m |
| [Backup Verification](./backup-verify.md) | @dba | 2026-03-01 | 20m |
| [Server Patching](./server-patching.md) | @platform | 2026-03-20 | 60m |
```

### Step 30: Slack Command for Quick Runbook Access

```bash
# Post this to #ops when an incident starts
/runbooks incident service-restart
# Returns link to runbook + last tested date
```

Create a simple slash command webhook that queries your runbook index.

### Step 31: Template 4: High Traffic / Scaling Response

```markdown
# High Traffic Response Runbook

**Owner:** @sre-team
**Last tested:** 2026-03-01
**Estimated time:** 15 minutes
**Severity:** Critical

### Step 32: Purpose
Scale production to handle traffic spikes without service degradation.

## Prerequisites
- Access to AWS Console or `kubectl` with production context
- Grafana dashboard: "Service Health > Request Rate"
- Confirm this is a real traffic spike, not a metrics scrape bug

### Step 33: Steps

### Kubernetes — Horizontal Scaling

1. Check current pod count and CPU/memory:
   ```bash
   kubectl get hpa -n production
   kubectl top pods -n production -l app=your-service
   ```

2. Manually scale if HPA is not triggering fast enough:
   ```bash
   kubectl scale deployment your-service -n production --replicas=10
   ```

3. Verify new pods start healthy:
   ```bash
   kubectl rollout status deployment/your-service -n production
   kubectl get pods -n production -l app=your-service
   ```

### Database — Connection Pool Check

1. Check Postgres connection count:
   ```bash
   psql -U postgres -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
   ```
 If active connections > 80% of max_connections: enable PgBouncer connection pooling.

2. Enable read replicas for read-heavy traffic:
   ```bash
   # Point reporting queries to replica
   export DB_READ_HOST=db-replica-01.example.com
   ```

### CDN / Cache

1. Check cache hit rate in Cloudflare dashboard
2. Purge stale cache if serving outdated content:
   ```bash
   curl -X POST "https://api.cloudflare.com/client/v4/zones/${CF_ZONE_ID}/purge_cache" \
     -H "Authorization: Bearer ${CF_API_TOKEN}" \
     -H "Content-Type: application/json" \
     --data '{"purge_everything":true}'
   ```

### Step 34: Verification

Traffic is handled when:
- Error rate < 0.5% (Grafana: Service Health > Error Rate)
- P95 response time < 500ms
- Pod CPU usage < 70% under load

### Step 35: Rollback / Scale Down

After traffic returns to normal (monitor for 30 minutes):
```bash
# Let HPA handle it, or manually scale back
kubectl scale deployment your-service -n production --replicas=3
```

### Step 36: Escalation

Traffic still unmanageable after 20 minutes: page @infrastructure-lead and open a Cloudflare support ticket if CDN appears to be the bottleneck.
```

### Step 37: Making Runbooks Findable at 3am

A runbook nobody can find in an incident is useless. Three places every runbook must live:

**1. The repo (source of truth):**
```
runbooks/
 incident/
 service-restart.md
 db-failover.md
 high-traffic.md
 deployments/
 deploy-hotfix.md
 rollback.md
 maintenance/
 ssl-renewal.md
 backup-verify.md
 server-patching.md
```

**2. Your internal docs tool** (Notion, Confluence, or a static site built from the same markdown). Mirror the repo structure exactly so links in Slack messages to runbooks do not break when people navigate around the docs site.

**3. Pinned in `#incidents`:**
Post a pinned message at the top of your incidents Slack channel with direct links to the five most-used runbooks. During an incident, people do not have time to navigate a wiki — the link should be one click away.

A runbook library only works if the team trusts it. Trust comes from: commands that actually run without modification, time estimates that are close to reality, and rollback steps that have actually been tested. Each time you use a runbook in a real incident, update the `Last tested` field and fix anything that was inaccurate. This feedback loop — use it, fix it, trust it more — is what separates a living runbook from documentation theatre.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [How to Write Runbooks for Remote Engineering Teams](/remote-work-tools/how-to-write-runbooks-remote-engineering-teams/)
- [Best Practice for Remote Team Escalation Paths](/remote-work-tools/best-practice-for-remote-team-escalation-paths-that-scale-wi/)
- [Best Practices for Remote Incident Communication](/remote-work-tools/best-practices-for-remote-incident-communication/)

---

## Related Articles

- [How to Build a Remote Team Runbook Library 2026](/remote-work-tools/how-to-build-remote-team-runbook-library-2026/)
- [How to Organize Remote Team Runbook Documentation for](/remote-work-tools/how-to-organize-remote-team-runbook-documentation-for-on-cal/)
- [Remote Team Charter Template Guide 2026](/remote-work-tools/remote-team-charter-template-guide-2026/)
- [How to Create Remote Team Playbook Templates](/remote-work-tools/how-to-create-remote-team-playbook-templates/)
- [How to Write Runbooks for Remote Engineering Teams](/remote-work-tools/how-to-write-runbooks-remote-engineering-teams/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)
```
{% endraw %}
