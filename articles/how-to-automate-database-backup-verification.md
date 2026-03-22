---
layout: default
title: "How to Automate Database Backup Verification"
description: "Automate PostgreSQL and MySQL backup verification with restore tests, row count checks, and alerting so backup integrity is proven not assumed"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-database-backup-verification/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A backup you've never restored is a hope, not a backup. Most teams find out their backups are broken during the disaster they were supposed to prevent. Automated verification runs the restore process on a schedule, validates the data, and alerts you before you need it.

---

## The Verification Strategy

For each backup, a complete verification cycle does:
1. Restore the backup to a temporary database
2. Run structural checks (tables exist, row counts are sane)
3. Run data integrity checks (referential integrity, no nulls where expected)
4. Record the result and timing
5. Drop the temporary database
6. Alert if anything fails

---

## PostgreSQL Backup Verification Script

```bash
#!/bin/bash
# scripts/verify-pg-backup.sh
# Verifies a PostgreSQL backup dump by restoring it and running checks

set -euo pipefail

# Configuration
BACKUP_S3_PATH="${1:-s3://your-backups/postgres/latest.dump}"
PG_HOST="${PG_HOST:-localhost}"
PG_PORT="${PG_PORT:-5432}"
PG_SUPERUSER="${PG_SUPERUSER:-postgres}"
TEST_DB="backup_verify_$(date +%s)"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
MIN_ROWS_USERS="${MIN_ROWS_USERS:-1000}"
LOG_FILE="/var/log/backup-verify/$(date +%Y%m%d_%H%M%S).log"

mkdir -p "$(dirname "$LOG_FILE")"

log() { echo "[$(date +%Y-%m-%dT%H:%M:%S)] $*" | tee -a "$LOG_FILE"; }
alert() {
  log "ALERT: $*"
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\":\"BACKUP VERIFY FAILED: $*\"}" \
      "$SLACK_WEBHOOK"
  fi
}

cleanup() {
  log "Cleaning up test database $TEST_DB"
  psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_SUPERUSER" \
    -c "DROP DATABASE IF EXISTS $TEST_DB;" postgres || true
}
trap cleanup EXIT

# Step 1: Download backup
log "Downloading backup from $BACKUP_S3_PATH"
START=$(date +%s)
aws s3 cp "$BACKUP_S3_PATH" /tmp/backup.dump
DOWNLOAD_TIME=$(($(date +%s) - START))
log "Download complete in ${DOWNLOAD_TIME}s ($(du -sh /tmp/backup.dump | cut -f1))"

# Step 2: Create test database
log "Creating test database: $TEST_DB"
psql -h "$PG_HOST" -p "$PG_PORT" -U "$PG_SUPERUSER" \
  -c "CREATE DATABASE $TEST_DB;" postgres

# Step 3: Restore
log "Restoring backup..."
START=$(date +%s)
pg_restore \
  -h "$PG_HOST" -p "$PG_PORT" -U "$PG_SUPERUSER" \
  -d "$TEST_DB" \
  --no-owner --no-acl \
  /tmp/backup.dump 2>&1 | tee -a "$LOG_FILE"
RESTORE_TIME=$(($(date +%s) - START))
log "Restore complete in ${RESTORE_TIME}s"

# Step 4: Run verification checks
PSQL="psql -h $PG_HOST -p $PG_PORT -U $PG_SUPERUSER -d $TEST_DB -t --no-align"

# Check critical tables exist
for table in users accounts transactions audit_log; do
  count=$($PSQL -c "SELECT COUNT(*) FROM $table" 2>/dev/null || echo "ERROR")
  if [ "$count" = "ERROR" ]; then
    alert "Table $table missing or query failed in backup restore"
    exit 1
  fi
  log "Table $table: $count rows"
done

# Check minimum row counts
user_count=$($PSQL -c "SELECT COUNT(*) FROM users")
if [ "$user_count" -lt "$MIN_ROWS_USERS" ]; then
  alert "users table has only $user_count rows (expected >= $MIN_ROWS_USERS). Backup may be truncated."
  exit 1
fi

# Check referential integrity (no orphaned records)
orphan_count=$($PSQL -c "
  SELECT COUNT(*) FROM transactions t
  LEFT JOIN users u ON t.user_id = u.id
  WHERE u.id IS NULL
")
if [ "$orphan_count" -gt "0" ]; then
  alert "Found $orphan_count orphaned transaction records in backup"
  exit 1
fi

# Check data recency (newest record should be recent)
latest_record=$($PSQL -c "SELECT MAX(created_at) FROM transactions")
log "Most recent transaction in backup: $latest_record"

# Compare with expected recency (should be within 25 hours)
expected_cutoff=$(date -d "25 hours ago" +%Y-%m-%dT%H:%M:%S 2>/dev/null \
  || date -v-25H +%Y-%m-%dT%H:%M:%S)
if [[ "$latest_record" < "$expected_cutoff" ]]; then
  alert "Backup data appears stale. Latest record: $latest_record"
  exit 1
fi

# Step 5: Record success metrics
log "SUCCESS: Backup verified"
log "Download: ${DOWNLOAD_TIME}s | Restore: ${RESTORE_TIME}s | Users: $user_count"

# Send success metric to monitoring
if command -v curl &>/dev/null; then
  curl -s -X POST "https://push.monitoring.yourcompany.com/metrics/job/backup_verify" \
    --data-binary "backup_restore_duration_seconds $RESTORE_TIME
backup_verify_success 1
backup_user_count $user_count" || true
fi

rm -f /tmp/backup.dump
log "Verification complete"
```

---

## Schedule in Cron

```bash
# /etc/cron.d/backup-verification
# Run daily at 5am UTC, verify previous night's backup
0 5 * * * postgres \
  SLACK_WEBHOOK="https://hooks.slack.com/..." \
  PG_HOST="localhost" \
  /opt/scripts/verify-pg-backup.sh \
  "s3://your-backups/postgres/$(date -d yesterday +%Y%m%d).dump" \
  >> /var/log/backup-verify/cron.log 2>&1
```

---

## GitHub Actions for Scheduled Verification

For teams without a dedicated ops server, run verification in GitHub Actions:

```yaml
# .github/workflows/backup-verify.yml
name: Database Backup Verification

on:
  schedule:
    - cron: '0 5 * * *'  # Daily at 5am UTC
  workflow_dispatch:

jobs:
  verify-backup:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: verify_target
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Download latest backup
        run: |
          LATEST=$(aws s3 ls s3://your-backups/postgres/ \
            | sort | tail -1 | awk '{print $4}')
          aws s3 cp "s3://your-backups/postgres/$LATEST" /tmp/backup.dump
          echo "BACKUP_FILE=/tmp/backup.dump" >> $GITHUB_ENV
          echo "BACKUP_NAME=$LATEST" >> $GITHUB_ENV

      - name: Restore backup
        run: |
          pg_restore \
            -h localhost -U postgres \
            -d verify_target \
            --no-owner --no-acl \
            $BACKUP_FILE
        env:
          PGPASSWORD: testpassword

      - name: Run verification checks
        run: |
          psql -h localhost -U postgres -d verify_target \
            -f scripts/verify-backup-checks.sql
        env:
          PGPASSWORD: testpassword

      - name: Notify on failure
        if: failure()
        run: |
          curl -s -X POST \
            -H 'Content-type: application/json' \
            --data "{\"text\":\"BACKUP VERIFY FAILED: ${{ env.BACKUP_NAME }}\"}" \
            ${{ secrets.SLACK_WEBHOOK }}
```

Create `scripts/verify-backup-checks.sql`:

```sql
-- verify-backup-checks.sql
-- All queries should return 0 to pass

\echo 'Checking critical tables exist...'
SELECT COUNT(*) as must_be_positive FROM users;
SELECT COUNT(*) as must_be_positive FROM accounts;
SELECT COUNT(*) as must_be_positive FROM transactions;

\echo 'Checking referential integrity...'
DO $$
DECLARE
  orphans INTEGER;
BEGIN
  SELECT COUNT(*) INTO orphans
  FROM transactions t
  LEFT JOIN users u ON t.user_id = u.id
  WHERE u.id IS NULL;

  IF orphans > 0 THEN
    RAISE EXCEPTION 'Found % orphaned transactions', orphans;
  END IF;
END $$;

\echo 'Checking data recency...'
DO $$
DECLARE
  latest TIMESTAMP;
BEGIN
  SELECT MAX(created_at) INTO latest FROM transactions;
  IF latest < NOW() - INTERVAL '25 hours' THEN
    RAISE EXCEPTION 'Backup data too old: latest record at %', latest;
  END IF;
END $$;

\echo 'All checks passed.'
```

---

## MySQL Backup Verification

For MySQL/MariaDB:

```bash
#!/bin/bash
# scripts/verify-mysql-backup.sh
set -euo pipefail

BACKUP_S3_PATH="${1:-s3://your-backups/mysql/latest.sql.gz}"
MYSQL_HOST="${MYSQL_HOST:-localhost}"
MYSQL_USER="${MYSQL_USER:-root}"
TEST_DB="backup_verify_$(date +%s)"

cleanup() {
  mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" \
    -e "DROP DATABASE IF EXISTS $TEST_DB;" 2>/dev/null || true
}
trap cleanup EXIT

# Download and restore
aws s3 cp "$BACKUP_S3_PATH" - \
  | gunzip \
  | mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" "$TEST_DB"

# Run checks
mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" "$TEST_DB" <<'SQL'
SELECT 'users check' AS test, COUNT(*) AS row_count FROM users;
SELECT 'transactions check' AS test, COUNT(*) AS row_count FROM transactions;

-- Fail if 0 rows
SELECT IF(COUNT(*) > 0, 'PASS', SIGNAL SQLSTATE '45000') FROM users;
SQL

echo "MySQL backup verified successfully"
```

---

## Related Reading

- [Setting Up pgBouncer for Connection Pooling](/remote-work-tools/setting-up-pgbouncer-for-connection-pooling/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Automate Infrastructure Cost Alerts](/remote-work-tools/how-to-automate-infrastructure-cost-alerts/)

- [Automate Expense Reports for Remote Workers](/remote-work-tools/automate-expense-reports-remote-workers/)
---

## Related Articles

- [Backblaze vs CrashPlan for Remote Work Backup](/remote-work-tools/backblaze-vs-crashplan-for-remote-work-backup/)
- [Remote Work Backup Strategy for Developers](/remote-work-tools/remote-work-backup-strategy-for-developers/)
- [How to Set Up Reliable Backup Internet for Remote Work](/remote-work-tools/how-to-set-up-reliable-backup-internet-for-remote-work-failover-guide/)
- [Best Backup Solutions for Remote Developer Machines](/remote-work-tools/best-backup-solutions-for-remote-developer-machines/)
- [Remote Work Internet Backup Solutions Comparison](/remote-work-tools/remote-work-internet-backup-solutions-comparison/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
