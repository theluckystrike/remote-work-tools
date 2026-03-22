---
layout: default
title: "How to Set Up Nginx Unit for App Deployment"
description: "Deploy Python, Node.js, and Go apps with Nginx Unit using its REST API for zero-downtime config changes, TLS termination, and process isolation on remote infra"
date: 2026-03-22
author: theluckystrike
permalink: /nginx-unit-app-deployment-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up Nginx Unit for App Deployment

Nginx Unit is an application server that handles multiple languages (Python, Node.js, Go, PHP, Ruby) through a single unified REST API. Unlike traditional Nginx, you reconfigure it with HTTP calls instead of editing files and reloading — which means zero-downtime deployments become a single `curl` command.

This guide covers installation, deploying Python and Node.js apps, TLS termination, routing multiple apps, Go deployment, process tuning, CI/CD integration via SSH tunnel, health checks, and logging.

---

## Why Nginx Unit Instead of a Traditional App Server

The standard deployment stack for Python or Node.js usually involves Gunicorn or PM2 sitting behind Nginx with a proxy_pass configuration. That works, but it means maintaining two separate configuration systems (Nginx config files and the app server config), two separate reload mechanisms, and two separate places where deployments can break.

Nginx Unit collapses this into one system with one configuration API. Some specific advantages:

- **API-driven config**: Deployments are HTTP PUT requests to a Unix socket. No file editing, no `systemctl reload`, no process restarts that drop connections.
- **Language isolation**: Each app runs in its own process pool with its own user. A memory leak in one Python app doesn't affect the Node.js app on the same host.
- **Atomic path swaps**: Point Unit at a new directory and in-flight requests finish against the old code while new requests go to the new code. No brief window of 502 errors.
- **No glue code**: Unit handles process management, signal handling, logging, and TLS. You do not need to write a systemd service for each app.

The tradeoff: Unit's community is smaller than Gunicorn's or PM2's. When something breaks, the debugging path is less well-documented. This guide includes a troubleshooting section to cover the common cases.

---

## Installation

```bash
# Ubuntu 22.04 / 24.04
curl --output /usr/share/keyrings/nginx-keyring.gpg \
  https://unit.nginx.org/keys/nginx-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/nginx-keyring.gpg] \
  https://packages.nginx.org/unit/ubuntu/ $(lsb_release -cs) unit" \
  > /etc/apt/sources.list.d/unit.list

sudo apt update
sudo apt install unit unit-python3.12 unit-nodejs unit-go

sudo systemctl enable unit && sudo systemctl start unit
```

Verify Unit is running:

```bash
sudo curl --unix-socket /var/run/control.unit.sock http://localhost/
# Returns current config as JSON
```

The `unit-python3.12`, `unit-nodejs`, and `unit-go` packages install language-specific modules. Install only the modules you need — each adds a small binary that Unit loads dynamically.

---

## Core Concepts

Unit uses a JSON config tree with three main sections:

- **listeners**: TCP/Unix sockets that accept connections
- **routes**: Request routing rules (path matching, headers)
- **applications**: Process pools for each app

All changes go through the control API socket. No files to edit, no service restarts.

The config is persistent — Unit stores its state and restores it on restart. You do not need to re-apply your config after a host reboot.

---

## Deploying a Python (FastAPI) App

```bash
sudo mkdir -p /var/www/fastapi-app
sudo chown unit:unit /var/www/fastapi-app

python3.12 -m venv /var/www/fastapi-app/venv
/var/www/fastapi-app/venv/bin/pip install fastapi uvicorn[standard]

cat > /var/www/fastapi-app/main.py << 'EOF'
from fastapi import FastAPI
app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/")
def root():
    return {"message": "Hello from Unit"}
EOF
```

Push the Unit configuration:

```bash
curl -X PUT --unix-socket /var/run/control.unit.sock http://localhost/config \
  -H "Content-Type: application/json" \
  -d '{
    "listeners": {
      "*:8000": {
        "pass": "applications/fastapi"
      }
    },
    "applications": {
      "fastapi": {
        "type": "python 3.12",
        "path": "/var/www/fastapi-app",
        "home": "/var/www/fastapi-app/venv",
        "module": "main",
        "callable": "app",
        "processes": {
          "max": 8,
          "spare": 2,
          "idle_timeout": 20
        }
      }
    }
  }'
```

Test it:

```bash
curl http://localhost:8000/health
# {"status":"ok"}
```

The `processes` block sets dynamic scaling. Unit starts with `spare` processes and scales up to `max` under load, then scales back down after `idle_timeout` seconds.

---

## Deploying a Node.js App

```bash
cat > /var/www/nodeapp/server.js << 'EOF'
const http = require("http");
const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ status: "ok", pid: process.pid }));
});
module.exports = server;
EOF

curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/nodeapp \
  -H "Content-Type: application/json" \
  -d '{
    "type": "node",
    "executable": "/usr/bin/node",
    "main": "/var/www/nodeapp/server.js",
    "processes": 4,
    "user": "unit",
    "group": "unit"
  }'

curl -X PUT --unix-socket /var/run/control.unit.sock \
  'http://localhost/config/listeners/*:8001' \
  -H "Content-Type: application/json" \
  -d '{"pass": "applications/nodeapp"}'
```

For Express apps, the `main` file should export the Express `app` object rather than call `app.listen()`. Unit manages the listener — calling `listen()` inside the app causes a conflict:

```javascript
// express-app.js
const express = require("express");
const app = express();

app.get("/health", (req, res) => res.json({ status: "ok" }));

module.exports = app; // export, do NOT call app.listen()
```

---

## Deploying a Go App

Go apps compile to a static binary that Unit runs as an external application. No language module is required:

```go
// main.go
package main

import (
    "encoding/json"
    "net/http"
)

func main() {
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
    })
    http.ListenAndServe(":0", nil)
}
```

```bash
go build -o /var/www/goapp/server /var/www/goapp/

curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/goapp \
  -H "Content-Type: application/json" \
  -d '{
    "type": "external",
    "executable": "/var/www/goapp/server",
    "processes": 2,
    "user": "unit",
    "group": "unit"
  }'

curl -X PUT --unix-socket /var/run/control.unit.sock \
  'http://localhost/config/listeners/*:8002' \
  -H "Content-Type: application/json" \
  -d '{"pass": "applications/goapp"}'
```

---

## TLS Termination

```bash
cat /etc/letsencrypt/live/example.com/fullchain.pem \
    /etc/letsencrypt/live/example.com/privkey.pem > /tmp/bundle.pem

curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/certificates/example-com \
  --data-binary @/tmp/bundle.pem

rm /tmp/bundle.pem

curl -X PUT --unix-socket /var/run/control.unit.sock \
  'http://localhost/config/listeners/*:443' \
  -H "Content-Type: application/json" \
  -d '{
    "pass": "applications/fastapi",
    "tls": {
      "certificate": "example-com",
      "protocols": ["TLSv1.2", "TLSv1.3"]
    }
  }'
```

Certificate rotation does not require a restart. Upload a new certificate bundle and update the listener — Unit swaps the certificate in-place:

```bash
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/certificates/example-com \
  --data-binary @/tmp/new-bundle.pem
```

---

## Zero-Downtime Deployments

```bash
# Deploy new code to a new path
rsync -a --delete build/ /var/www/fastapi-app-v2/

# Atomic swap via Unit config — no requests dropped
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/fastapi/path \
  -H "Content-Type: application/json" \
  -d '"/var/www/fastapi-app-v2"'

# Check application status
curl --unix-socket /var/run/control.unit.sock \
  http://localhost/status/applications/fastapi/
```

Unit processes in-flight requests with the old code, then starts new requests against the new path. There is no connection drop.

To roll back, swap the path back:

```bash
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/fastapi/path \
  -H "Content-Type: application/json" \
  -d '"/var/www/fastapi-app"'
```

---

## Routing Multiple Apps on One Port

```bash
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config \
  -H "Content-Type: application/json" \
  -d '{
    "listeners": {
      "*:80": {
        "pass": "routes"
      }
    },
    "routes": [
      {
        "match": { "uri": "/api/*" },
        "action": { "pass": "applications/fastapi" }
      },
      {
        "match": { "uri": "/app/*" },
        "action": { "pass": "applications/nodeapp" }
      },
      {
        "action": { "share": "/var/www/static/$uri" }
      }
    ]
  }'
```

Routes can also match on request headers, HTTP method, or source IP. This replaces most Nginx location block logic without needing a separate proxy server.

---

## Exposing the Control API for CI/CD

Never expose Unit's control socket directly over the network — it has no authentication layer. Tunnel from the CI runner:

```bash
# Tunnel from CI runner to the Unix socket
ssh -L 9000:/var/run/control.unit.sock user@prod-host -N &

# Deploy from CI using the tunnel
curl -X PUT http://localhost:9000/config/applications/fastapi/path \
  -H "Content-Type: application/json" \
  -d '"/var/www/fastapi-app-v3"'
```

**Full GitHub Actions deployment job:**

```yaml
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to Unit via SSH tunnel
      run: |
        mkdir -p ~/.ssh
        echo "${{ secrets.DEPLOY_KEY }}" > ~/.ssh/deploy_key
        chmod 600 ~/.ssh/deploy_key
        ssh -i ~/.ssh/deploy_key -o StrictHostKeyChecking=no \
          -L 9000:/var/run/control.unit.sock \
          deploy@prod-host -N &
        sleep 2
        rsync -az --delete dist/ deploy@prod-host:/var/www/fastapi-app-v${{ github.sha }}/
        curl -sf -X PUT http://localhost:9000/config/applications/fastapi/path \
          -H "Content-Type: application/json" \
          -d '"/var/www/fastapi-app-v${{ github.sha }}"'
```

---

## Systemd Service

```ini
# /etc/systemd/system/unit.service
[Unit]
Description=NGINX Unit
After=network.target

[Service]
Type=forking
PIDFile=/var/run/unit/unit.pid
ExecStartPre=/usr/share/doc/unit/examples/check_config.sh
ExecStart=/usr/sbin/unitd --log /var/log/unit.log --pid /var/run/unit/unit.pid
ExecStop=/bin/kill -QUIT $MAINPID
KillMode=mixed
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## Troubleshooting Common Issues

**"Unable to apply new configuration" with no detail**: Check `/var/log/unit.log` for the actual error. Unit's API response is terse but the log file has the full stack trace.

**App returns 502 immediately**: The application module may not be installed. Verify with `dpkg -l | grep unit` that `unit-python3.12` (or the relevant module) is installed.

**Python app can't find packages**: Ensure the `home` field points to the virtualenv directory, not the site-packages directory. Unit infers the Python path from the virtualenv.

**Port conflicts**: If another process is already listening on the port, Unit logs a bind error. Check with `ss -tlnp | grep :8000`.

**Permission denied on Unix socket**: The control socket at `/var/run/control.unit.sock` is owned by root by default. Run `curl` with `sudo`, or add your user to the group that owns the socket.

---

## Related Reading

- [How to Set Up Keel for Continuous Delivery](/remote-work-tools/keel-continuous-delivery-setup/)
- [How to Create Automated Rollback Systems](/remote-work-tools/automated-rollback-systems/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
