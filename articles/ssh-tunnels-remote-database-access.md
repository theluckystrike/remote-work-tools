---
layout: default
title: "SSH Tunnels for Remote Database Access"
description: "Set up SSH tunnels to securely access remote databases without exposing ports. Covers local forwarding, dynamic SOCKS, autossh, and GUI tool configs."
date: 2026-03-21
last_modified_at: 2026-03-21
author: theluckystrike
permalink: /ssh-tunnels-remote-database-access/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Exposing database ports directly to the internet is a security risk. SSH tunnels let you access remote databases as if they were running locally — all traffic is encrypted through SSH, and the database port never needs to be opened in your firewall.

This guide covers local port forwarding for databases, jump hosts, persistent tunnels with autossh, and configuring GUI database tools to use them.

## How SSH Local Port Forwarding Works

A local port forward binds a port on your machine and tunnels all traffic through SSH to a destination:

```
[your machine :5433] → [SSH to server] → [server :5432 (Postgres)]
```

The basic syntax:

```bash
# ssh -L [local-port]:[remote-host]:[remote-port] [ssh-host]
ssh -L 5433:localhost:5432 user@db-server.example.com

# Now connect to Postgres locally on port 5433
psql -h 127.0.0.1 -p 5433 -U myuser -d mydb
```

The `-L` flag means local port forward. Port 5433 on your machine now routes to port 5432 on `db-server.example.com` (where `localhost` means the server itself).

## Common Database Tunnels

### PostgreSQL

```bash
# Standard tunnel
ssh -L 5433:localhost:5432 ubuntu@db.example.com -N

# -N means don't execute a remote command — just forward
# -f means run in background (combine with -N)
ssh -fN -L 5433:localhost:5432 ubuntu@db.example.com

# Connect via tunnel
psql -h 127.0.0.1 -p 5433 -U appuser -d production

# Or with URL
DATABASE_URL=postgresql://appuser:password@127.0.0.1:5433/production psql
```

### MySQL / MariaDB

```bash
ssh -fN -L 3307:localhost:3306 ubuntu@db.example.com

mysql -h 127.0.0.1 -P 3307 -u appuser -p mydatabase
```

### Redis

```bash
ssh -fN -L 6380:localhost:6379 ubuntu@cache.example.com

redis-cli -h 127.0.0.1 -p 6380 ping
```

### MongoDB

```bash
ssh -fN -L 27018:localhost:27017 ubuntu@mongo.example.com

mongosh "mongodb://127.0.0.1:27018/mydb"
```

## Database on a Private Network (Jump Host)

When the database is on a private network and only reachable through a bastion/jump host:

```bash
# Database at 10.0.1.50:5432, only reachable from bastion
# Bastion is at bastion.example.com

# Single command with -J (jump host)
ssh -fN -L 5433:10.0.1.50:5432 -J ubuntu@bastion.example.com ubuntu@10.0.1.50

# Or with ProxyJump in ~/.ssh/config
```

Configure `~/.ssh/config` to make this permanent:

```bash
# ~/.ssh/config

Host bastion
    HostName bastion.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host db-private
    HostName 10.0.1.50
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion

# Then tunnel through the configured host
ssh -fN -L 5433:localhost:5432 db-private
```

## Persistent Tunnels with autossh

Plain `ssh -fN` tunnels die when the connection drops. `autossh` monitors the tunnel and restarts it automatically:

```bash
# Install autossh
sudo apt-get install autossh   # Debian/Ubuntu
brew install autossh            # macOS

# Start a persistent tunnel
autossh -M 20000 -fN -L 5433:localhost:5432 ubuntu@db.example.com

# -M 20000 sets the monitoring port (autossh sends keepalives here)
# -fN: background, no command

# Disable autossh's own keepalive and use SSH's instead
AUTOSSH_GATETIME=0 autossh -M 0 -fN \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -L 5433:localhost:5432 ubuntu@db.example.com
```

### Run autossh as a systemd service

```bash
# /etc/systemd/system/ssh-tunnel-db.service
sudo tee /etc/systemd/system/ssh-tunnel-db.service > /dev/null << 'EOF'
[Unit]
Description=SSH Tunnel to Production Database
After=network-online.target
Wants=network-online.target

[Service]
User=ubuntu
ExecStart=/usr/bin/autossh -M 0 -N \
  -o "ServerAliveInterval=30" \
  -o "ServerAliveCountMax=3" \
  -o "ExitOnForwardFailure=yes" \
  -o "StrictHostKeyChecking=no" \
  -i /home/ubuntu/.ssh/id_ed25519 \
  -L 5433:localhost:5432 \
  ubuntu@db.example.com
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ssh-tunnel-db
sudo systemctl start ssh-tunnel-db
sudo systemctl status ssh-tunnel-db
```

## Shell Aliases for Quick Tunnel Management

```bash
# Add to ~/.bashrc or ~/.zshrc

# Start tunnels
alias tunnel-db='autossh -M 0 -fN -o "ServerAliveInterval 30" -L 5433:localhost:5432 ubuntu@db.example.com'
alias tunnel-redis='autossh -M 0 -fN -o "ServerAliveInterval 30" -L 6380:localhost:6379 ubuntu@cache.example.com'
alias tunnel-all='tunnel-db && tunnel-redis && echo "Tunnels started"'

# Kill all SSH tunnels
alias tunnel-kill='pkill -f "ssh.*-fN" && echo "All tunnels killed"'

# Check active tunnels
alias tunnel-list='ps aux | grep "ssh.*-fN" | grep -v grep'

# Check if a port is in use
alias port-check='lsof -ti'
```

## Configure GUI Database Tools

### TablePlus

1. New Connection → PostgreSQL
2. Host: `127.0.0.1`
3. Port: `5433` (your local tunnel port)
4. Database: your database name
5. User/Password: your credentials

TablePlus also has built-in SSH tunnel support: Connection → SSH → enable, fill in server details. This is equivalent to starting the tunnel manually.

### DBeaver

1. New Database Connection → PostgreSQL
2. In connection dialog, go to `SSH` tab
3. Enable SSH Tunnel
4. Host: `db.example.com`, Port: `22`
5. Auth Method: Public Key, Private Key: path to `~/.ssh/id_ed25519`
6. Main tab: Host: `localhost`, Port: `5432`

### pgAdmin 4

```json
// pgAdmin SSH tunnel config (via GUI)
// Servers → Create → Server
// Connection tab:
//   Host: 127.0.0.1
//   Port: 5433
//   Username: appuser
// SSH Tunnel tab:
//   Enable SSH tunneling: yes
//   Tunnel host: db.example.com
//   Tunnel port: 22
//   Username: ubuntu
//   Authentication: Identity file
//   Identity file: /home/user/.ssh/id_ed25519
```

### DataGrip

JetBrains DataGrip handles SSH tunnels natively in the data source configuration. In the Data Sources panel, select your data source, open the SSH/SSL tab, and check "Use SSH tunnel." Specify the SSH host, port 22, and your private key. DataGrip maintains the tunnel for the lifetime of the IDE session and reconnects automatically if the SSH connection drops. This is one of the most reliable built-in tunnel implementations for GUI clients.

## Using SSH Tunnels with ORMs and Application Code

For development environments, you often want your application code to connect through a tunnel rather than configuring firewall rules. A clean pattern is a tunnel-aware connection wrapper:

```python
# tunnel_db.py — start an SSH tunnel before connecting
import subprocess
import time
import psycopg2

def start_tunnel(ssh_host, remote_db_port, local_port):
    """Start an SSH tunnel and return the subprocess."""
    proc = subprocess.Popen([
        'ssh', '-fN',
        '-L', f'{local_port}:localhost:{remote_db_port}',
        '-o', 'StrictHostKeyChecking=no',
        '-o', 'ServerAliveInterval=30',
        ssh_host
    ])
    time.sleep(1)  # Brief wait for tunnel to establish
    return proc

# Usage: tunnel to dev DB before running migrations
tunnel = start_tunnel('ubuntu@db.staging.example.com', 5432, 5433)
conn = psycopg2.connect(
    host='127.0.0.1',
    port=5433,
    database='mydb',
    user='appuser',
    password='secret'
)
```

For Rails applications, the `sshtunnel` gem provides equivalent functionality and integrates cleanly with `database.yml`. Node.js projects can use the `tunnel-ssh` npm package, which wraps the same SSH forwarding logic.

## Tunnel Comparison: Manual SSH vs autossh vs GUI Built-in

| Method | Best For | Reconnects | Extra Deps | Persistent |
|--------|----------|-----------|-----------|-----------|
| `ssh -fN` | One-off queries | No | None | No |
| `autossh` | Developer machines | Yes | autossh | No (unless systemd) |
| systemd + autossh | Server/CI environments | Yes | autossh + systemd | Yes |
| GUI built-in (TablePlus, DBeaver) | GUI-only workflows | Varies | GUI client | Session only |
| DataGrip SSH | JetBrains shops | Yes (IDE session) | DataGrip | Session only |

For local development on a Mac, the `autossh` alias approach offers the best balance of simplicity and reliability. For CI/CD pipelines or staging servers that need permanent database access, the systemd service is the correct choice.

## Verify and Debug Tunnels

```bash
# Check if tunnel port is listening locally
lsof -i :5433
# or
ss -tlnp | grep 5433
# or
netstat -tlnp | grep 5433

# Test connection through tunnel
nc -zv 127.0.0.1 5433

# Verbose SSH connection for debugging
ssh -vvv -L 5433:localhost:5432 ubuntu@db.example.com

# Common errors and fixes:
# "bind: Address already in use" — another tunnel already on that port
lsof -ti:5433 | xargs kill  # kill whatever is using port 5433

# "channel 3: open failed: connect failed"
# The remote host can't reach the destination (firewall or wrong address)
# Test on the remote server: telnet localhost 5432
```

A useful diagnostic when the tunnel establishes but database connections fail: SSH to the remote server directly and attempt `psql -h localhost -p 5432`. If that fails, the issue is on the remote server (database not listening, pg_hba.conf blocking local connections), not the tunnel itself.



## Frequently Asked Questions


**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.


**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.


**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.


**How do I get my team to adopt a new tool?**

Start with a small pilot group of willing early adopters. Let them use it for 2-3 weeks, then gather their honest feedback. Address concerns before rolling out to the full team. Forced adoption without buy-in almost always fails.


**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.


## Related Articles

- [teleport-db-config.yaml](/remote-work-tools/how-to-secure-remote-team-database-access-with-just-in-time-/)
- [Best SSH Key Management Solution for Distributed Remote](/remote-work-tools/best-ssh-key-management-solution-for-distributed-remote-engi/)
- [Remote Team Runbook Template for Database Failover](/remote-work-tools/remote-team-runbook-template-for-database-failover-procedure/)
- [Notion Database Templates for a Solo Recruiter Working Remot](/remote-work-tools/notion-database-templates-for-a-solo-recruiter-working-remot/)
- [Best Cloud Access Security Broker for Remote Teams Using](/remote-work-tools/best-cloud-access-security-broker-for-remote-teams-using-multiple-saas/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
