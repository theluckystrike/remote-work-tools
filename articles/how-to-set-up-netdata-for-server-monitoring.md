---
layout: default
title: "How to Set Up Netdata for Server Monitoring"
description: "Install and configure Netdata for real-time server monitoring with alerting, dashboards, and remote team access for distributed infrastructure"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-netdata-for-server-monitoring/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Netdata gives you per-second metrics across CPU, memory, disk, network, and hundreds of application plugins — with zero configuration for most use cases. For remote teams managing distributed infrastructure, it cuts mean time to detect from minutes to seconds.

This guide covers installation, configuration, alerting, and exposing Netdata securely for remote access.

---

## Install Netdata

The quickest path on Debian/Ubuntu:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh
sh /tmp/netdata-kickstart.sh --stable-channel --disable-telemetry
```

For RHEL/Rocky/Alma:

```bash
wget -O /tmp/netdata-kickstart.sh https://my-netdata.io/kickstart.sh
sh /tmp/netdata-kickstart.sh --stable-channel --disable-telemetry --dont-start-it
systemctl enable netdata && systemctl start netdata
```

Docker deployment for containers-first teams:

```yaml
# docker-compose.yml
version: "3.8"
services:
  netdata:
    image: netdata/netdata:stable
    container_name: netdata
    pid: host
    network_mode: host
    restart: unless-stopped
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      - netdataconfig:/etc/netdata
      - netdatalib:/var/lib/netdata
      - netdatacache:/var/cache/netdata
      - /:/host/root:ro,rslave
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
    environment:
      - NETDATA_CLAIM_TOKEN=your_claim_token
      - NETDATA_CLAIM_URL=https://app.netdata.cloud
      - NETDATA_CLAIM_ROOMS=your_room_id

volumes:
  netdataconfig:
  netdatalib:
  netdatacache:
```

---

## Core Configuration

Netdata's main config is at `/etc/netdata/netdata.conf`. Edit it with the helper:

```bash
/etc/netdata/edit-config netdata.conf
```

Key settings to tune:

```ini
[global]
    # How many seconds of data to keep in RAM (default: 3600)
    history = 7200

    # Update frequency in seconds (1 = per-second metrics)
    update every = 1

    # Memory mode: ram, map, save, dbengine
    memory mode = dbengine

[web]
    # Bind only to localhost if using a reverse proxy
    bind to = 127.0.0.1
    port = 19999

[plugins]
    # Disable plugins you don't need to reduce overhead
    fping = no
    xenstat = no
```

Set the hostname so dashboards are readable when managing multiple servers:

```bash
# /etc/netdata/netdata.conf
[global]
    hostname = prod-web-01
```

---

## Configure Health Alerts

Netdata ships with hundreds of built-in alerts. Override them in `/etc/netdata/health.d/`. Create a custom alert file:

```bash
/etc/netdata/edit-config health.d/custom-disk.conf
```

```ini
# Alert when disk space on / exceeds 80%
alarm: disk_space_root
     on: disk.space
  hosts: *
 lookup: average -1m percentage of used
  units: %
  every: 1m
   warn: $this > 80
   crit: $this > 90
   info: Disk usage on root filesystem is high
     to: sysadmin

# Alert when load average exceeds CPU count * 2
alarm: load_15
     on: system.load
  hosts: *
 lookup: average -15m sum of load15
  units: load
  every: 1m
   warn: $this > (($system_cores is nan) ? 4 : ($system_cores * 2))
   crit: $this > (($system_cores is nan) ? 8 : ($system_cores * 4))
   info: 15-minute load average is very high
     to: sysadmin
```

---

## Set Up Email and Slack Alerts

Configure notification channels at `/etc/netdata/health_alarm_notify.conf`:

```bash
/etc/netdata/edit-config health_alarm_notify.conf
```

For email:

```bash
# Enable email notifications
SEND_EMAIL="YES"
DEFAULT_RECIPIENT_EMAIL="ops@yourcompany.com"

# Use sendmail or postfix
sendmail="/usr/sbin/sendmail"
```

For Slack:

```bash
SEND_SLACK="YES"
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/T.../B.../..."
DEFAULT_RECIPIENT_SLACK="#alerts"

# Optionally route critical to a different channel
role_recipients_slack[sysadmin]="#ops-critical"
```

Test the notification setup:

```bash
sudo -u netdata /usr/libexec/netdata/plugins.d/alarm-notify.sh test
```

---

## Enable Persistent Storage (dbengine)

By default Netdata uses RAM for storage. For longer retention, use the database engine:

```ini
# /etc/netdata/netdata.conf
[global]
    memory mode = dbengine

[db]
    # Tiers define retention at different resolutions
    # Tier 0: per-second data
    dbengine multihost disk space MB = 1024

    # Tier 1: per-minute averages (auto-managed)
    dbengine tier 1 multihost disk space MB = 512
```

Check how much space Netdata is using:

```bash
du -sh /var/cache/netdata/dbengine/
```

---

## Expose Dashboard Securely with Nginx

Never expose Netdata's port 19999 directly. Put it behind Nginx with basic auth or SSO:

```nginx
# /etc/nginx/sites-available/netdata
upstream netdata {
    server 127.0.0.1:19999;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name monitoring.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/monitoring.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/monitoring.yourcompany.com/privkey.pem;

    auth_basic "Netdata Monitoring";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://netdata;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for Netdata's streaming protocol
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Create the htpasswd file:

```bash
htpasswd -c /etc/nginx/.htpasswd netdata-user
nginx -t && systemctl reload nginx
```

---

## Stream Metrics to a Parent Node

For remote teams with multiple servers, stream all metrics to one parent Netdata instance rather than maintaining N dashboards.

On each child node (`/etc/netdata/stream.conf`):

```ini
[stream]
    enabled = yes
    destination = parent-netdata.yourcompany.com:19999
    api key = aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
    timeout seconds = 60
    buffer size bytes = 1048576
    reconnect delay seconds = 5
    initial clock resync iterations = 60
```

On the parent node (`/etc/netdata/stream.conf`):

```ini
[aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee]
    enabled = yes
    # Where to store data for this child
    default memory mode = dbengine
    health enabled by default = auto
    allow from = *
```

Restart both nodes:

```bash
systemctl restart netdata
```

The parent dashboard at `https://monitoring.yourcompany.com` now shows all hosts in the left sidebar.

---

## Monitor Docker Containers

Netdata auto-detects Docker if the socket is accessible. For the Docker deployment, the socket is already mounted. For bare-metal installs:

```bash
usermod -aG docker netdata
systemctl restart netdata
```

Verify containers appear:

```bash
curl -s http://127.0.0.1:19999/api/v1/charts | python3 -m json.tool | grep '"id":.*docker'
```

To limit which containers are tracked, edit `/etc/netdata/docker.conf`:

```ini
[plugin:cgroups]
    # Disable cgroup memory accounting (reduces overhead)
    enable memory limits in alarms = no

    # Whitelist containers by name pattern
    # Leave empty to track all
    cgroup include only = prod-*
```

---

## Netdata Cloud for Team Dashboards

Netdata Cloud (free tier available) lets remote teams share dashboards without managing access to individual servers.

```bash
# Claim a node to Netdata Cloud
netdata-claim.sh \
  -token=YOUR_CLAIM_TOKEN \
  -rooms=YOUR_ROOM_ID \
  -url=https://app.netdata.cloud
```

Get your claim token from: **Netdata Cloud > Space Settings > Nodes > Connect Nodes**

Once claimed, the node appears in your cloud space and is accessible to all team members with room access — no VPN or SSH tunnels required.

---

## Useful Diagnostic Commands

```bash
# Check Netdata service status
systemctl status netdata

# View live logs
journalctl -u netdata -f

# Test a specific plugin
sudo -u netdata /usr/libexec/netdata/charts.d/postgres.chart.sh

# Check what's consuming Netdata CPU
netdata-claim.sh --check-claiming-status

# Dump all collected metrics to JSON
curl -s "http://127.0.0.1:19999/api/v1/allmetrics?format=json" | jq '.'

# List all active alarms
curl -s "http://127.0.0.1:19999/api/v1/alarms?all" | jq '.alarms | to_entries[] | .value | {name, status, value}'
```

---

## Related Reading

- [How to Set Up Traefik Reverse Proxy](/remote-work-tools/how-to-set-up-traefik-reverse-proxy/)
- [How to Automate Database Backup Verification](/remote-work-tools/how-to-automate-database-backup-verification/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
