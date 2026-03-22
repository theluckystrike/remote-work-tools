---
layout: default
title: "Setting Up Harbor for Container Registry"
description: "Deploy Harbor as a self-hosted container registry with image scanning, replication, LDAP auth, and robot accounts for remote DevOps teams"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-harbor-for-container-registry/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Harbor is an open-source container registry that goes beyond basic storage: built-in Trivy image scanning, replication to cloud registries, robot accounts for CI, LDAP/OIDC auth, and a web UI. Remote teams get one registry their entire pipeline can trust, with audit logs showing who pushed what.

## Key Takeaways

- **Harbor is an open-source**: container registry that goes beyond basic storage: built-in Trivy image scanning, replication to cloud registries, robot accounts for CI, LDAP/OIDC auth, and a web UI.
- **Topics covered**: prerequisites, installation, configuration
- **Practical guidance included**: Step-by-step setup and configuration instructions
- **Use-case recommendations**: Specific guidance based on team size and requirements

## Prerequisites

- Docker and Docker Compose installed
- Domain with HTTPS cert (or use Harbor's built-in cert generation)
- 4 vCPU, 8GB RAM, 40GB+ disk

## Installation

```bash
# Download Harbor installer
HARBOR_VERSION="v2.10.0"
wget "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/harbor-online-installer-${HARBOR_VERSION}.tgz"
tar xzvf "harbor-online-installer-${HARBOR_VERSION}.tgz"
cd harbor
```

## Configuration

```yaml
# harbor.yml
hostname: registry.example.com

https:
  port: 443
  certificate: /etc/letsencrypt/live/registry.example.com/fullchain.pem
  private_key: /etc/letsencrypt/live/registry.example.com/privkey.pem

harbor_admin_password: your-strong-admin-password

database:
  password: your-db-password
  max_idle_conns: 100
  max_open_conns: 900

data_volume: /data/harbor

trivy:
  ignore_unfixed: false
  skip_update: false
  offline_scan: false
  insecure: false
  github_token: ""
  timeout: 5m0s
  skip_db_update: false

jobservice:
  max_job_workers: 10

notification:
  webhook_job_max_retry: 10

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200m
    location: /var/log/harbor

_version: 2.10.0
```

```bash
# Install
sudo ./install.sh --with-trivy

# Check status
docker compose -f /path/to/harbor/docker-compose.yml ps
```

## Nginx Frontend (if using existing nginx)

```nginx
server {
    listen 443 ssl http2;
    server_name registry.example.com;

    ssl_certificate /etc/letsencrypt/live/registry.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/registry.example.com/privkey.pem;

    client_max_body_size 0;  # No limit for large images
    chunked_transfer_encoding on;

    location / {
        proxy_pass https://localhost:8443;  # Harbor's HTTPS port
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}
```

## OIDC Authentication (Keycloak)

```
Harbor Admin UI > Administration > Configuration > Authentication
  Auth Mode: OIDC Provider
  OIDC Provider Name: Company SSO
  OIDC Endpoint: https://auth.example.com/realms/company
  OIDC Client ID: harbor
  OIDC Client Secret: your-client-secret
  OIDC Scope: openid,email,profile,groups
  Group Claim Name: groups
  OIDC Admin Group: harbor-admins
  Verify Certificate: true
  Auto Onboard: true
  Username Claim: preferred_username
```

## Project Structure

```bash
# Create projects via Harbor CLI (harbor-cli) or API
curl -X POST "https://registry.example.com/api/v2.0/projects" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "project_name": "production",
    "metadata": {
      "public": "false",
      "enable_content_trust": "true",
      "prevent_vul": "true",
      "severity": "high",
      "auto_scan": "true"
    }
  }'

# Create projects for each environment
for project in production staging development shared-libs; do
  curl -X POST "https://registry.example.com/api/v2.0/projects" \
    -H "Content-Type: application/json" \
    -u "admin:your-admin-password" \
    -d "{\"project_name\": \"${project}\", \"metadata\": {\"public\": \"false\", \"auto_scan\": \"true\"}}"
done
```

## Robot Accounts for CI/CD

```bash
# Create robot account for CI pipeline (project-scoped)
curl -X POST "https://registry.example.com/api/v2.0/projects/production/robots" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "name": "ci-robot",
    "description": "CI/CD pipeline robot account",
    "duration": 365,
    "access": [
      {"resource": "repository", "action": "pull"},
      {"resource": "repository", "action": "push"},
      {"resource": "artifact", "action": "delete"}
    ]
  }'
# Save the returned token — it only appears once!
```

```yaml
# GitHub Actions using robot account
# .github/workflows/build.yml
- name: Login to Harbor
  uses: docker/login-action@v3
  with:
    registry: registry.example.com
    username: ${{ secrets.HARBOR_ROBOT_NAME }}
    password: ${{ secrets.HARBOR_ROBOT_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: registry.example.com/production/my-app:${{ github.sha }}
```

## Image Scanning Policies

```bash
# Enable auto-scan on push for a project
curl -X PUT "https://registry.example.com/api/v2.0/projects/production" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "metadata": {
      "auto_scan": "true",
      "severity": "high",
      "prevent_vul": "true"
    }
  }'

# Trigger manual scan
curl -X POST "https://registry.example.com/api/v2.0/projects/production/repositories/my-app/artifacts/sha256:abc123/scan" \
  -u "admin:your-admin-password"

# Get scan results
curl -s "https://registry.example.com/api/v2.0/projects/production/repositories/my-app/artifacts/sha256:abc123/additions/vulnerabilities" \
  -u "admin:your-admin-password" | jq '.[] | {severity, description: .vulnerabilities[].description}' | head -20
```

## Replication to AWS ECR

```bash
# Add AWS ECR endpoint as replication target
curl -X POST "https://registry.example.com/api/v2.0/registries" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "name": "aws-ecr-us-east-1",
    "type": "aws-ecr",
    "url": "https://123456789.dkr.ecr.us-east-1.amazonaws.com",
    "access_key": "YOUR_AWS_ACCESS_KEY",
    "access_secret": "YOUR_AWS_SECRET_KEY",
    "insecure": false
  }'

# Create replication rule: push production to ECR on push
curl -X POST "https://registry.example.com/api/v2.0/replication/policies" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "name": "sync-to-ecr",
    "src_registry": {"id": 0},
    "dest_registry": {"id": 1},
    "dest_namespace": "production",
    "filters": [
      {"type": "name", "value": "production/**"},
      {"type": "tag", "value": "v*"}
    ],
    "trigger": {"type": "event_based", "trigger_settings": {"event_types": ["PUSH"]}},
    "enabled": true
  }'
```

## Daily Garbage Collection

```bash
# Schedule GC via Harbor admin UI:
# Administration > Garbage Collection > GC Settings
# Schedule: Daily at 02:00 UTC

# Or trigger manually
curl -X POST "https://registry.example.com/api/v2.0/system/gc/schedule" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{"schedule": {"type": "Manual"}}'
```

## Pull Images

```bash
# Login
docker login registry.example.com
# Username: alice (or robot account)
# Password: your-password or token

# Pull
docker pull registry.example.com/production/my-app:v1.2.3

# Kubernetes: create imagePullSecret
kubectl create secret docker-registry harbor-secret \
  --docker-server=registry.example.com \
  --docker-username=robot$ci-robot \
  --docker-password=your-robot-token \
  --namespace=production
```

## Related Reading

- [How to Set Up Kubernetes Dev Cluster Remotely](/remote-work-tools/how-to-set-up-kubernetes-dev-cluster-remotely/)
- [Best Container Registry Tool for Remote Teams](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Setting Up Keycloak for Team SSO](/remote-work-tools/setting-up-keycloak-for-team-sso/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
