---
layout: default
title: "Setting Up pgBouncer for Connection Pooling"
description: "Install and configure pgBouncer to reduce PostgreSQL connection overhead, improve throughput, and support high-concurrency workloads for remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-pgbouncer-for-connection-pooling/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

PostgreSQL creates a new OS process for each connection. At 500 concurrent connections, you're running 500 processes — and memory usage, context switching, and lock contention all scale with that count. pgBouncer sits in front of PostgreSQL and multiplexes thousands of application connections onto a small pool of real database connections.

For remote teams running microservices, each service having its own connection pool means you can hit hundreds of connections fast. pgBouncer is the standard fix.

---

## Install pgBouncer

On Debian/Ubuntu:

```bash
apt-get install pgbouncer
```

Docker:

```yaml
# docker-compose.yml
services:
  pgbouncer:
    image: edoburu/pgbouncer:1.22.1
    environment:
      - DB_USER=app_user
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_HOST=postgres
      - DB_NAME=myapp
      - POOL_MODE=transaction
      - MAX_CLIENT_CONN=1000
      - DEFAULT_POOL_SIZE=25
      - AUTH_TYPE=scram-sha-256
    ports:
      - "5432:5432"
    depends_on:
      - postgres
```

---

## Core Configuration

The main config file is `/etc/pgbouncer/pgbouncer.ini`:

```ini
[databases]
; Syntax: alias = host=... port=... dbname=... user=...
myapp = host=localhost port=5432 dbname=myapp
myapp_readonly = host=replica.yourcompany.internal port=5432 dbname=myapp user=readonly_user

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

; Pooling mode:
; session    - one server conn per client session (safest, least efficient)
; transaction - one server conn per transaction (recommended)
; statement  - one server conn per statement (only for autocommit)
pool_mode = transaction

; Max client connections pgBouncer accepts
max_client_conn = 1000

; Default connections per user+database pair
default_pool_size = 25

; Minimum connections to keep alive per pool
min_pool_size = 5

; How long to wait for a free connection before error
pool_timeout = 30

; Idle server connection lifetime
server_idle_timeout = 600

; Client idle connection lifetime
client_idle_timeout = 0

; TCP keepalive
tcp_keepalive = 1
tcp_keepcnt = 9
tcp_keepidle = 300

; Admin interface
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

; Log settings
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

; PID file
pidfile = /var/run/postgresql/pgbouncer.pid

; Unix socket for local connections
unix_socket_dir = /var/run/postgresql
```

---

## Authentication Setup

pgBouncer uses `userlist.txt` for authentication. Generate entries with a helper script:

```bash
#!/bin/bash
# scripts/pgbouncer-add-user.sh
# Usage: ./pgbouncer-add-user.sh username password

USER=$1
PASS=$2
USERLIST="/etc/pgbouncer/userlist.txt"

# Generate scram-sha-256 hash
HASH=$(psql -c "SELECT concat('\"SCRAM-SHA-256\"\$', encode(digest('$PASS', 'sha256'), 'base64'), '\$')" \
  -t --no-align)

# Or use md5 (simpler, less secure)
MD5_HASH=$(echo -n "${PASS}${USER}" | md5sum | cut -d' ' -f1)
echo "\"$USER\" \"md5${MD5_HASH}\"" >> "$USERLIST"
echo "User $USER added to pgBouncer userlist"
```

For scram-sha-256 (recommended for PostgreSQL 14+), use pg_dumpall to extract the auth string:

```bash
psql -U postgres -t -A -c \
  "SELECT concat('\"', rolname, '\" \"', rolpassword, '\"') FROM pg_authid WHERE rolpassword IS NOT NULL;" \
  > /etc/pgbouncer/userlist.txt
```

Add a cron job to refresh the userlist when users change:

```bash
# /etc/cron.d/pgbouncer-userlist
*/5 * * * * postgres psql -U postgres -t -A -c \
  "SELECT concat('\"', rolname, '\" \"', rolpassword, '\"') FROM pg_authid WHERE rolpassword IS NOT NULL;" \
  > /etc/pgbouncer/userlist.txt && \
  psql -h /var/run/postgresql -p 6432 -U pgbouncer_admin pgbouncer -c "RELOAD;"
```

---

## Per-Database Pool Configuration

Set different pool sizes for different databases or users:

```ini
[databases]
; High-traffic OLTP database — large pool
myapp = host=localhost port=5432 dbname=myapp pool_size=50

; Analytics database — smaller pool, longer timeout acceptable
analytics = host=localhost port=5432 dbname=analytics pool_size=10 pool_timeout=60

; Read replica for read-heavy services
myapp_ro = host=replica.db.internal port=5432 dbname=myapp user=readonly pool_size=30
```

Override pool settings per user:

```ini
[users]
api_service = pool_mode=transaction pool_size=30
worker_service = pool_mode=session pool_size=5
```

---

## Monitor Pool Stats

Connect to the pgBouncer admin console:

```bash
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer
```

Key queries:

```sql
-- Current pool state
SHOW POOLS;
-- Shows: database, user, cl_active, cl_waiting, sv_active, sv_idle, sv_used

-- Connected clients
SHOW CLIENTS;

-- Server connections
SHOW SERVERS;

-- Statistics (requests, bytes, avg query time)
SHOW STATS;

-- Configuration
SHOW CONFIG;

-- Reload config without restart
RELOAD;

-- Gracefully kill idle connections
KILL myapp;
```

Automate monitoring with a script:

```bash
#!/bin/bash
# scripts/pgbouncer-stats.sh
PSQL="psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer -t --no-align"

echo "=== Pool Usage ==="
$PSQL -c "SHOW POOLS;" | column -t -s '|'

echo ""
echo "=== Waiting Clients ==="
$PSQL -c "SELECT database, user, cl_waiting FROM pools WHERE cl_waiting > 0;" \
  | column -t -s '|'

echo ""
echo "=== Query Rates (last 1s) ==="
$PSQL -c "SELECT database, total_xact_count, avg_xact_time FROM stats;" \
  | column -t -s '|'
```

---

## Integrate with Application Code

The application connects to pgBouncer instead of PostgreSQL directly. No code changes needed — just update the connection string:

```bash
# Before (direct PostgreSQL)
DATABASE_URL=postgresql://app_user:password@db.yourcompany.com:5432/myapp

# After (through pgBouncer)
DATABASE_URL=postgresql://app_user:password@pgbouncer.yourcompany.com:6432/myapp
```

For transaction mode (the most common), avoid connection-level features that don't survive across transactions:
- `SET search_path` — use schema-qualified table names instead
- `LISTEN/NOTIFY` — use session mode for these
- Prepared statements — disable in your driver

Disable prepared statements in common drivers:

```python
# Python / asyncpg
conn = await asyncpg.connect(
    dsn="postgresql://user:pass@pgbouncer:6432/myapp",
    statement_cache_size=0  # Disable prepared statements
)
```

```go
// Go / pgx
config, _ := pgx.ParseConfig("postgres://user:pass@pgbouncer:6432/myapp")
config.DefaultQueryExecMode = pgx.QueryExecModeSimpleProtocol
```

```javascript
// Node.js / node-postgres
const pool = new Pool({
  connectionString: 'postgresql://user:pass@pgbouncer:6432/myapp',
  statement_timeout: 30000,
  // pg doesn't use prepared statements by default in Pool mode
});
```

---

## Sizing Your Pool

The optimal pool size is not "as large as possible." PostgreSQL's throughput peaks at a specific concurrency level based on CPU cores. A common formula:

```
pool_size = (num_cpu_cores * 2) + num_disk_spindles
```

For a 4-core server with SSD:

```
pool_size = (4 * 2) + 1 = 9-16 is reasonable
```

Monitor `sv_idle` in `SHOW POOLS`. If idle servers are consistently > 20% of pool size, reduce pool size. If `cl_waiting` is ever nonzero for more than a few seconds, increase pool size or add read replicas.

---

## Related Reading

- [How to Automate Database Backup Verification](/remote-work-tools/how-to-automate-database-backup-verification/)
- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [Setting Up Consul for Service Discovery](/remote-work-tools/setting-up-consul-for-service-discovery/)

- [Best Goal Setting Framework Tool for Remote Teams Using OKRs](/remote-work-tools/best-goal-setting-framework-tool-for-remote-teams-using-okrs/)
---

## Related Articles

- [How to Build a Remote Team Runbook Library 2026](/remote-work-tools/how-to-build-remote-team-runbook-library-2026/)
- [Setting Up Consul for Service Discovery](/remote-work-tools/setting-up-consul-for-service-discovery/)
- [Secure File Transfer Protocol Setup for Remote Teams](/remote-work-tools/secure-file-transfer-protocol-setup-for-remote-teams-exchang/)
- [Setting Up a Remote Dev Server with Hetzner](/remote-work-tools/setting-up-remote-dev-server-with-hetzner/)
- [How to Set Up Ansible for Remote Server Management](/remote-work-tools/how-to-set-up-ansible-remote-server-management/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
