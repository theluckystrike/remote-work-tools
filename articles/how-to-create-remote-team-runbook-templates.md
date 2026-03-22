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

## Key Takeaways

- **Create test database: ```bash**: createdb -U postgres test_restore_$(date +%Y%m%d) ``` 4.
- **Restore backup: ```bash DB_NAME="test_restore_$(date**: +%Y%m%d)" gunzip -c /tmp/test-restore.sql.gz | psql -U postgres "$DB_NAME" ``` 5.
- **Restart with rolling update**: (preferred): ```bash kubectl rollout restart deployment/your-service -n production ``` 4.
- **Topics covered**: runbook structure standard, purpose, prerequisites

## Runbook Structure Standard

Every runbook must have these sections:

```markdown
# [Operation Name] Runbook

**Owner:** @team-name
**Last tested:** YYYY-MM-DD
**Estimated time:** N minutes
**Severity:** Critical / High / Medium / Low

## Purpose
One sentence: what does this runbook do?

## Prerequisites
- Access required
- Tools needed
- Checks to do first

## Steps
1. Step with command
2. Step with expected output
3. Verification step

## Verification
How to confirm the operation succeeded.

## Rollback
How to undo this if something goes wrong.

## Escalation
Who to page if this doesn't work.
```

## Template 1: Service Restart

```markdown
# Service Restart Runbook

**Owner:** @platform-team
**Last tested:** 2026-03-15
**Estimated time:** 5 minutes
**Severity:** High

## Purpose
Safely restart a production service without extended downtime.

## Prerequisites
- SSH access to production servers
- Confirm: `kubectl get pods -n production` (for k8s) or SSH access
- Alert #incidents that restart is in progress

## Steps

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

## Verification

```bash
# Check service responds
curl -s --max-time 10 https://api.example.com/health | jq .
# Expected: {"status": "ok"}

# Check error rate in Grafana:
# Dashboard: Service Health > Error Rate > last 5 minutes
# Expected: < 0.1% errors
```

## Rollback

If the service doesn't come back up:

```bash
# Kubernetes: rollback to previous version
kubectl rollout undo deployment/your-service -n production
kubectl rollout status deployment/your-service -n production

# Docker: start previous container
docker start your-service_previous
```

## Escalation

Service still down after 10 minutes: page @on-call-engineer via PagerDuty.
```

## Template 2: Database Backup Verification

```markdown
# Database Backup Verification Runbook

**Owner:** @database-team
**Last tested:** 2026-03-01
**Estimated time:** 20 minutes
**Severity:** Medium

## Purpose
Verify that recent database backup is valid and can be restored.

## Prerequisites
- Access to backup storage (S3/MinIO)
- Test restore environment available
- At least 10GB free disk space on test host

## Steps

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

## Cleanup

```bash
dropdb -U postgres "test_restore_$(date +%Y%m%d)"
rm /tmp/test-restore.sql.gz
```

## Verification

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

## Escalation

Backup older than 36 hours or restore fails: page @database-team immediately.
```

## Template 3: SSL Certificate Renewal

```markdown
# SSL Certificate Renewal Runbook

**Owner:** @platform-team
**Last tested:** 2026-01-10
**Estimated time:** 15 minutes (automated) / 45 minutes (manual)

## Purpose
Renew SSL certificates before expiry. Run this 30 days before expiry.

## Prerequisites
- Root/sudo access to servers running nginx/apache
- Certbot installed, or access to certificate provider dashboard

## Check Current Expiry

```bash
# Check all certs on a server
for domain in api.example.com git.example.com auth.example.com; do
  echo -n "$domain: "
  echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -dates 2>/dev/null | grep notAfter
done
```

## Automated Renewal (Let's Encrypt)

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

## Manual Renewal (Other CA)

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

## Verification

```bash
# Verify new expiry date
echo | openssl s_client -servername api.example.com \
  -connect api.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates
# notAfter should be 90 days from now (Let's Encrypt) or per CA
```
```

## Runbook CI — Auto-Test Commands

Test runbook commands don't drift from reality:

```yaml
# .github/workflows/test-runbooks.yml
name: Test Runbook Commands

on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday
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

## Runbook Index Template

```markdown
# Runbook Index

## Incident Response
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [Service Restart](./service-restart.md) | @platform | 2026-03-15 | 5m |
| [Database Failover](./db-failover.md) | @dba | 2026-02-01 | 30m |
| [High Traffic Response](./high-traffic.md) | @sre | 2026-03-01 | 15m |

## Deployments
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [Deploy Hotfix](./deploy-hotfix.md) | @engineering | 2026-03-10 | 20m |
| [Rollback Release](./rollback.md) | @engineering | 2026-03-05 | 10m |

## Maintenance
| Runbook | Owner | Last Tested | Time |
|---------|-------|-------------|------|
| [SSL Renewal](./ssl-renewal.md) | @platform | 2026-01-10 | 15m |
| [Backup Verification](./backup-verify.md) | @dba | 2026-03-01 | 20m |
| [Server Patching](./server-patching.md) | @platform | 2026-03-20 | 60m |
```

## Slack Command for Quick Runbook Access

```bash
# Post this to #ops when an incident starts
/runbooks incident service-restart
# Returns link to runbook + last tested date
```

Create a simple slash command webhook that queries your runbook index.

## Related Reading

- [How to Write Runbooks for Remote Engineering Teams](/remote-work-tools/how-to-write-runbooks-remote-engineering-teams/)
- [Best Practice for Remote Team Escalation Paths](/remote-work-tools/best-practice-for-remote-team-escalation-paths-that-scale-wi/)
- [Best Practices for Remote Incident Communication](/remote-work-tools/best-practices-for-remote-incident-communication/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
