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

---

## Core Concepts

Unit uses a JSON config tree with three main sections:

- **listeners**: TCP/Unix sockets that accept connections
- **routes**: Request routing rules (path matching, headers)
- **applications**: Process pools for each app

All changes go through the control API socket. No files to edit, no service restarts.

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

---

## Exposing the Control API for CI/CD

```bash
# Tunnel from CI runner to the Unix socket
ssh -L 9000:/var/run/control.unit.sock user@prod-host -N &

# Deploy from CI using the tunnel
curl -X PUT http://localhost:9000/config/applications/fastapi/path \
  -H "Content-Type: application/json" \
  -d '"/var/www/fastapi-app-v3"'
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

## Health Checks and Process Monitoring

Unit tracks application process state via the status endpoint. Poll it to confirm healthy startup before updating your load balancer:

```bash
# Wait for application to report running processes
check_unit_health() {
  local app="$1"
  local retries=10
  local delay=3

  for i in $(seq 1 $retries); do
    STATUS=$(curl -s --unix-socket /var/run/control.unit.sock \
      "http://localhost/status/applications/${app}/processes" 2>/dev/null)
    RUNNING=$(echo "$STATUS" | python3 -c "import sys,json; print(json.load(sys.stdin).get('running', 0))")
    if [ "${RUNNING:-0}" -gt 0 ]; then
      echo "App $app: $RUNNING process(es) running"
      return 0
    fi
    echo "Waiting for $app... (attempt $i/$retries)"
    sleep "$delay"
  done
  echo "ERROR: $app failed to start"
  return 1
}

check_unit_health fastapi
```

Unit log messages go to `/var/log/unit.log` — tail it during deployments to catch startup errors before they reach users:

```bash
# Watch for errors during deployment
tail -f /var/log/unit.log | grep -E "(error|warning|NOTICE)"

# Common startup errors:
# "failed to apply new conf" — JSON syntax error in config
# "unable to open ... module" — language module not installed
# "system error: permission denied" — wrong user/group for app files
```

For automated rollback, capture the current config before deploying and restore if the health check fails:

```bash
# Save current config
PREV_CONFIG=$(curl -s --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/fastapi)

# Deploy
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/fastapi/path \
  -d '"/var/www/fastapi-app-v3"'

# Health check
if ! check_unit_health fastapi; then
  echo "Rollback triggered"
  echo "$PREV_CONFIG" | curl -X PUT --unix-socket /var/run/control.unit.sock \
    http://localhost/config/applications/fastapi -H "Content-Type: application/json" -d @-
fi
```

## Go App Deployment

Unit supports Go apps compiled as shared libraries. Unlike Python or Node, Go apps need to be compiled with Unit's Go module:

```bash
# Install Go module for Unit
go get unit.nginx.org/go

# main.go — wrap your handler with Unit's ListenAndServe
package main

import (
    "fmt"
    "net/http"
    "unit.nginx.org/go"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, `{"status":"ok","path":"%s"}`, r.URL.Path)
}

func main() {
    http.HandleFunc("/", handler)
    unit.ListenAndServe(":0", nil)
}
```

```bash
# Build as a shared library
go build -buildmode=c-shared -o /var/www/goapp/app.so

# Configure Unit
curl -X PUT --unix-socket /var/run/control.unit.sock \
  http://localhost/config/applications/goapp \
  -H "Content-Type: application/json" \
  -d '{
    "type": "go",
    "executable": "/var/www/goapp/app.so",
    "processes": 4,
    "user": "unit",
    "group": "unit"
  }'
```

Go apps in Unit run as native shared libraries — no interpreter overhead, no port conflicts between apps. Each app uses Unit's shared process pool management regardless of language.

## Related Reading

- [How to Set Up Keel for Continuous Delivery](/remote-work-tools/keel-continuous-delivery-setup/)
- [How to Create Automated Rollback Systems](/remote-work-tools/automated-rollback-systems/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
