---
layout: default
title: "Setting Up Harbor for Container Registry"
description: "Deploy Harbor as a self-hosted container registry with image scanning, replication, LDAP auth, and robot accounts for remote DevOps teams"
date: 2026-03-22
author: theluckystrike
permalink: /setting-up-harbor-for-container-registry/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Harbor is an open-source container registry that goes beyond basic storage: built-in Trivy image scanning, replication to cloud registries, robot accounts for CI, LDAP/OIDC auth, and a web UI. Remote teams get one registry their entire pipeline can trust, with audit logs showing who pushed what.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Nginx Frontend (if using existing nginx)](#nginx-frontend-if-using-existing-nginx)
- [OIDC Authentication (Keycloak)](#oidc-authentication-keycloak)
- [Project Structure](#project-structure)
- [Robot Accounts for CI/CD](#robot-accounts-for-cicd)
- [Image Scanning Policies](#image-scanning-policies)
- [Replication to AWS ECR](#replication-to-aws-ecr)
- [Daily Garbage Collection](#daily-garbage-collection)
- [Pull Images](#pull-images)
- [Tag Retention Policies](#tag-retention-policies)
- [Webhook Notifications for Scan Results](#webhook-notifications-for-scan-results)
- [Backup Strategy](#backup-strategy)
- [Enforcing Content Trust with Cosign](#enforcing-content-trust-with-cosign)
- [Monitoring Harbor Health](#monitoring-harbor-health)
- [Related Reading](#related-reading)

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

## Tag Retention Policies

Unmanaged registries accumulate thousands of untagged image layers and stale feature-branch tags. Harbor's retention policies let you declaratively control what stays and what gets pruned.

```bash
# Create retention policy via API
curl -X POST "https://registry.example.com/api/v2.0/retentions" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "algorithm": "or",
    "rules": [
      {
        "priority": 1,
        "disabled": false,
        "action": "retain",
        "template": "latestPushedK",
        "params": {"latestPushedK": 10},
        "tag_selectors": [{"kind": "doublestar", "decoration": "matches", "pattern": "v*"}],
        "scope_selectors": {"repository": [{"kind": "doublestar", "decoration": "repoMatches", "pattern": "**"}]}
      },
      {
        "priority": 2,
        "disabled": false,
        "action": "retain",
        "template": "nDaysSinceLastPush",
        "params": {"nDaysSinceLastPush": 7},
        "tag_selectors": [{"kind": "doublestar", "decoration": "matches", "pattern": "main-*"}],
        "scope_selectors": {"repository": [{"kind": "doublestar", "decoration": "repoMatches", "pattern": "**"}]}
      }
    ],
    "scope": {"level": "project", "ref": 1},
    "trigger": {
      "kind": "Schedule",
      "settings": {"cron": "0 3 * * *"}
    }
  }'
```

This policy retains the 10 most recently pushed version-tagged images indefinitely and keeps `main-*` tags for 7 days. Everything else is eligible for garbage collection during the nightly GC run. Run GC after the retention job to actually reclaim disk space — retention only unlinks tags; GC deletes the blobs.

## Webhook Notifications for Scan Results

Harbor can fire webhooks on push, scan completion, and policy violations, making it straightforward to integrate with Slack or PagerDuty for security alerting:

```bash
# Create webhook for Slack notification on scan completion
curl -X POST "https://registry.example.com/api/v2.0/projects/production/webhook/policies" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{
    "name": "slack-scan-alerts",
    "description": "Notify Slack when image scan finds critical CVEs",
    "event_types": ["SCANNING_COMPLETED", "SCANNING_FAILED"],
    "targets": [
      {
        "type": "http",
        "address": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK",
        "auth_header": "",
        "skip_cert_verify": false
      }
    ],
    "enabled": true
  }'
```

The webhook payload includes the image name, tag, digest, scan status, and a URL to the vulnerability report in Harbor's UI. A lightweight AWS Lambda or Cloud Function can parse this payload and send a formatted Slack message with only the critical and high findings, avoiding notification fatigue from informational-level CVEs.

## Backup Strategy

Harbor's data lives in three places: the PostgreSQL database (project metadata, users, policies, replication rules), the Redis cache (session state, job queues), and the image blob storage under `data_volume`. A complete backup covers all three:

```bash
#!/bin/bash
# scripts/backup-harbor.sh
set -e
DATE=$(date +%Y%m%d_%H%M%S)
HARBOR_DIR="/opt/harbor"
BACKUP_DIR="/backups/harbor/${DATE}"
mkdir -p "$BACKUP_DIR"

# Stop Harbor gracefully (optional — for consistency)
# docker compose -f "${HARBOR_DIR}/docker-compose.yml" stop

# Dump PostgreSQL
docker exec harbor-db pg_dumpall -U postgres > "${BACKUP_DIR}/harbor-db.sql"

# Copy blob storage
rsync -a /data/harbor/registry/ "${BACKUP_DIR}/registry/"

# Copy config
cp "${HARBOR_DIR}/harbor.yml" "${BACKUP_DIR}/harbor.yml"

# Compress and ship
tar czf "/backups/harbor-${DATE}.tar.gz" -C "/backups/harbor" "${DATE}"
rm -rf "$BACKUP_DIR"

# Upload to S3
aws s3 cp "/backups/harbor-${DATE}.tar.gz" "s3://your-backup-bucket/harbor/"
echo "Harbor backup complete: harbor-${DATE}.tar.gz"
```

Restore by extracting the archive, restoring the database dump with `psql`, syncing the registry blobs back to `data_volume`, and restarting Harbor. Test restores quarterly — a backup you have never restored is a backup you cannot trust.

## Enforcing Content Trust with Cosign

Harbor supports Cosign signatures for supply chain security. After signing images with your CI pipeline's private key, Harbor can be configured to block pulls of unsigned images from the production project.

First, generate a cosign key pair and store the private key as a CI secret:

```bash
# Generate key pair
cosign generate-key-pair

# Keys are written to cosign.key (private) and cosign.pub (public)
# Add cosign.key as a CI/CD secret: COSIGN_PRIVATE_KEY
# Commit cosign.pub to your repo for verification
```

In your GitHub Actions build workflow, sign after push:

```yaml
- name: Build and push image
  id: build
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: registry.example.com/production/my-app:${{ github.sha }}

- name: Sign image with Cosign
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
    COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY \
      registry.example.com/production/my-app@${{ steps.build.outputs.digest }}
```

Enable content trust enforcement in Harbor at the project level:

```bash
curl -X PUT "https://registry.example.com/api/v2.0/projects/production" \
  -H "Content-Type: application/json" \
  -u "admin:your-admin-password" \
  -d '{"metadata": {"enable_content_trust_cosign": "true"}}'
```

With this enabled, Harbor blocks any `docker pull` or Kubernetes pull against the production project if the image digest has no valid Cosign signature. This is the most practical supply chain control available without a full Sigstore infrastructure.

## Monitoring Harbor Health

Harbor exposes a `/api/v2.0/health` endpoint that returns the status of each internal component (database, registry, jobservice, Redis, Trivy). Scrape it from your monitoring stack:

```bash
# Check health
curl -s "https://registry.example.com/api/v2.0/health" | jq '.components[] | select(.status != "healthy")'

# Prometheus scrape config for Harbor metrics
# Harbor exposes metrics at /metrics on the admin port (9090 by default)
```

```yaml
# prometheus.yml scrape job
scrape_configs:
  - job_name: harbor
    static_configs:
      - targets: ['registry.example.com:9090']
    metrics_path: /metrics
    scheme: https
    tls_config:
      insecure_skip_verify: false
```

Enable Harbor metrics in `harbor.yml` before installation:

```yaml
metric:
  enabled: true
  port: 9090
  path: /metrics
```

Key metrics to alert on: `harbor_project_artifact_total` (artifact count growth), `harbor_jobservice_job_total` with status `Error` (replication or scan job failures), and `harbor_registry_request_duration_seconds` for pull latency. A Grafana dashboard built on these three signals covers the most common operational failure modes without requiring deep Harbor expertise.

## Related Reading

- [How to Set Up Kubernetes Dev Cluster Remotely](/remote-work-tools/how-to-set-up-kubernetes-dev-cluster-remotely/)
- [Best Container Registry Tool for Remote Teams](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Setting Up Keycloak for Team SSO](/remote-work-tools/setting-up-keycloak-for-team-sso/)

---

## Related Articles

- [Best Container Registry Tool for Remote Teams Sharing](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Setting Up Keycloak for Team SSO](/remote-work-tools/setting-up-keycloak-for-team-sso/)
- [How to Set Up Verdaccio Private npm Registry](/remote-work-tools/how-to-set-up-verdaccio-private-npm-registry/)
- [How to Set Up Portainer for Docker Management](/remote-work-tools/how-to-set-up-portainer-for-docker-management/)
- [Setting Up Grafana Dashboards for Remote Teams](/remote-work-tools/setting-up-grafana-dashboards-for-remote-teams/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
