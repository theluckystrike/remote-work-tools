---
layout: default
title: "How to Set Up Teleport for Secure Access"
description: "Deploy Teleport to give remote teams zero-trust SSH, Kubernetes, and database access with short-lived certificates, audit logs, and no VPN required"
date: 2026-03-22
author: theluckystrike
permalink: /teleport-secure-access-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up Teleport for Secure Access

VPNs give remote engineers a network-level tunnel to everything, which means a compromised laptop gets network-level access to everything. Teleport does the opposite: every resource (SSH servers, Kubernetes clusters, databases) requires a short-lived certificate issued per-session, tied to the user's identity, with a full audit trail of every command run.

---

## Architecture Overview

```
Developer laptop
     │
     ▼
Teleport Proxy (public HTTPS endpoint)
     │
     ├── SSH nodes (Teleport agents)
     ├── Kubernetes API server
     ├── PostgreSQL / MySQL via Database Service
     └── Web apps via Application Service
```

Teleport Auth Service issues X.509 certificates valid for hours or days. Nothing is accessible without a current certificate. All sessions are recorded.

---

## Install the Teleport Cluster

**Option A: Single-node with Docker Compose (lab/small team)**

```yaml
# docker-compose.yml
version: "3.8"
services:
  teleport:
    image: public.ecr.aws/gravitational/teleport:16
    restart: unless-stopped
    command: teleport start --config=/etc/teleport/teleport.yaml
    volumes:
      - ./teleport.yaml:/etc/teleport/teleport.yaml:ro
      - teleport-data:/var/lib/teleport
    ports:
      - "443:443"
      - "3022:3022"   # SSH proxy
      - "3025:3025"   # Teleport nodes connect here

volumes:
  teleport-data:
```

**`teleport.yaml` (proxy + auth on one node):**

```yaml
version: v3
teleport:
  nodename: teleport.yourcompany.internal
  data_dir: /var/lib/teleport
  log:
    output: stderr
    severity: INFO

auth_service:
  enabled: true
  cluster_name: yourcompany
  tokens:
    - "node:${NODE_JOIN_TOKEN}"
    - "kube:${KUBE_JOIN_TOKEN}"
    - "db:${DB_JOIN_TOKEN}"

proxy_service:
  enabled: true
  web_listen_addr: 0.0.0.0:443
  public_addr: teleport.yourcompany.com:443
  ssh_public_addr: teleport.yourcompany.com:3023
  https_cert_file: /etc/teleport/tls.crt
  https_key_file: /etc/teleport/tls.key

ssh_service:
  enabled: false  # separate SSH nodes, not the auth node
```

---

## Add SSH Nodes

Install the Teleport agent on each server you want to access:

```bash
# Install on Ubuntu
curl https://deb.releases.teleport.dev/teleport-pubkey.asc | sudo gpg --dearmor -o /usr/share/keyrings/teleport-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/teleport-archive-keyring.gpg] \
  https://deb.releases.teleport.dev/ stable main" | sudo tee /etc/apt/sources.list.d/teleport.list
sudo apt update && sudo apt install teleport

# Configure as an SSH node joining your cluster
cat > /etc/teleport/teleport.yaml << EOF
version: v3
teleport:
  data_dir: /var/lib/teleport
  auth_token: "node:your-join-token"
  auth_server: teleport.yourcompany.com:3025

auth_service:
  enabled: false

proxy_service:
  enabled: false

ssh_service:
  enabled: true
  commands:
    - name: hostname
      command: [hostname]
      period: 1m
  labels:
    env: production
    region: us-east-1
    role: web
EOF

sudo systemctl enable teleport && sudo systemctl start teleport
```

Verify the node appears:

```bash
tctl nodes ls
# Node      Address          Labels
# web-01    10.0.1.10:3022   env=production,region=us-east-1,role=web
```

---

## User and Role Setup

Create a role that allows SSH to production nodes:

```yaml
# role-dev.yaml
kind: role
version: v6
metadata:
  name: dev
spec:
  allow:
    logins: ["ubuntu", "ec2-user"]
    node_labels:
      env: ["staging", "development"]
      # NOT production
    kubernetes_groups: ["developers"]
    db_names: ["*"]
    db_users: ["readonly"]
  deny:
    node_labels:
      env: ["production"]
  options:
    max_session_ttl: 8h
    require_session_mfa: false
```

```yaml
# role-sre.yaml
kind: role
version: v6
metadata:
  name: sre
spec:
  allow:
    logins: ["ubuntu", "root"]
    node_labels:
      "*": "*"   # all nodes
    kubernetes_groups: ["system:masters"]
    db_names: ["*"]
    db_users: ["admin", "readonly"]
  options:
    max_session_ttl: 12h
    require_session_mfa: true   # require MFA for production access
    record_session:
      default: best_effort
```

```bash
tctl create -f role-dev.yaml
tctl create -f role-sre.yaml

# Create a user
tctl users add alice --roles=dev --logins=ubuntu
# Outputs a one-time invite link
```

---

## Connecting from the Developer's Machine

```bash
# Install tsh (Teleport client)
brew install teleport  # macOS
# or download from https://goteleport.com/download/

# Log in
tsh login --proxy=teleport.yourcompany.com --user=alice

# List available servers
tsh ls
tsh ls env=production

# SSH into a node
tsh ssh ubuntu@web-01
tsh ssh --login=ubuntu ubuntu@web-01

# SSH with local port forwarding
tsh ssh -L 5432:localhost:5432 ubuntu@db-proxy-01

# Start an interactive Kubernetes session
tsh kube login my-cluster
kubectl get pods -A
```

---

## Database Access

Register a PostgreSQL database:

```yaml
# db-service config section in teleport.yaml on the database host
db_service:
  enabled: true
  databases:
    - name: prod-postgres
      protocol: postgres
      uri: localhost:5432
      static_labels:
        env: production
```

Connect from the developer's machine:

```bash
# Log into the database (issues a short-lived cert)
tsh db login prod-postgres --db-user=readonly --db-name=myapp

# Connect with psql
tsh db connect prod-postgres
# or
psql "$(tsh db env --format=uri prod-postgres)"
```

---

## Audit Log Review

Every SSH session, command, and query is logged to the audit log. Query it:

```bash
# View recent audit events
tctl audit log -format=json | jq '.'

# Find all commands run by alice in the last 24h
tctl audit log --type=session.command \
  --from=$(date -d "24 hours ago" +%Y-%m-%dT%H:%M:%SZ) \
  | jq 'select(.metadata.user == "alice")'

# Export session recording
tsh recordings ls
tsh play session-id-here
```

---

## GitHub Actions Integration

Use Teleport Machine ID for CI/CD access without static credentials:

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # for GitHub OIDC
    steps:
      - name: Fetch Teleport certificates
        uses: teleport-actions/auth@v2
        with:
          proxy: teleport.yourcompany.com:443
          token: github-actions-token  # Machine ID join token
          certificate-ttl: 1h

      - name: Deploy via SSH
        run: |
          tsh ssh ubuntu@web-01 'cd /opt/myapp && ./deploy.sh'
```

---

## Related Reading

- [Best Tools for Remote Team Secret Sharing](/remote-work-tools/remote-team-secret-sharing-tools/)
- [How to Create Automated Security Scan Pipelines](/remote-work-tools/automated-security-scan-pipelines/)
- [How to Set Up ArgoCD for GitOps Workflows](/remote-work-tools/argocd-gitops-workflow-setup/)
- [How to Setup Vpn Secure Remote Access Office Resources](/remote-work-tools/how-to-setup-vpn-secure-remote-access-office-resources/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
