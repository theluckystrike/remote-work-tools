---
layout: default
title: "How to Set Up Keel for Continuous Delivery"
description: "Deploy Keel in Kubernetes to automatically update Deployments and Helm releases when new container images are pushed — with approval workflows and Slack notifications"
date: 2026-03-22
author: theluckystrike
permalink: /keel-continuous-delivery-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up Keel for Continuous Delivery

Keel watches your container registries and automatically updates Kubernetes Deployments, StatefulSets, DaemonSets, and Helm releases when new images are published. Unlike ArgoCD's GitOps model, Keel operates in-cluster and responds directly to registry events — which makes it useful for rapid iteration on staging environments where you want every push to main to deploy immediately.

This guide covers Helm installation, per-workload update policies, Slack approval workflows, webhook triggers from GitHub Actions, Helm release updates, and Prometheus metrics for monitoring Keel's behavior.

---

## How Keel Works

Keel runs as a Deployment in your cluster. It monitors container registries (by polling or webhook), compares image tags against your running workloads, and updates them according to per-workload update policies defined in annotations or Helm values.

Update policies:
- `all`: update on any new tag
- `major`, `minor`, `patch`: semver-aware updates
- `force`: force-pull the same tag (useful for `latest`)
- `glob:prod-*`: update when a tag matching a glob is pushed

Keel requires the `keel.sh/policy` annotation (or Helm values equivalent) on every workload it manages. Workloads without an annotation are ignored entirely, so you can run Keel in a cluster that has both auto-managed and manually managed services without conflict.

---

## Install Keel with Helm

```bash
helm repo add keel https://charts.keel.sh
helm repo update

# Install with Slack notifications and approval enabled
cat > keel-values.yaml << 'EOF'
# Slack notifications
slack:
  enabled: true
  token: "xoxb-your-slack-bot-token"
  channel: "#deployments"
  approvalsChannel: "#infra-approvals"

# Enable Keel webhook endpoint for registry push triggers
webhooks:
  enabled: true

# RBAC — Keel needs to update deployments
rbac:
  enabled: true
  serviceAccount:
    create: true

# Helm provider to update Helm releases too
helmProvider:
  enabled: true
  version: v3

# Slack approval for production deployments
approvals:
  slack:
    enabled: true

# Poll interval for registries without webhooks
polling:
  defaultSchedule: "@every 3m"
EOF

helm upgrade --install keel keel/keel \
  -n keel \
  --create-namespace \
  -f keel-values.yaml
```

The Slack bot token needs `chat:write`, `channels:read`, and `reactions:read` scopes. The reactions scope is required for Keel to detect approval emoji responses in the `#infra-approvals` channel.

Verify Keel is running:

```bash
kubectl get pods -n keel
kubectl logs -n keel -l app=keel --tail=50
```

---

## Annotate Deployments for Auto-Update

Keel reads annotations on Deployments to know the update policy:

**Auto-update on any new semver minor/patch on staging:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: staging
  annotations:
    keel.sh/policy: minor          # update on minor and patch releases
    keel.sh/trigger: poll          # poll registry every 3 minutes
    keel.sh/notify: "#deployments" # Slack channel for notifications
    keel.sh/approvals: "0"         # no approval required for staging
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: ghcr.io/yourorg/myapp:1.2.0
```

**Require Slack approval for production:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  annotations:
    keel.sh/policy: minor
    keel.sh/trigger: poll
    keel.sh/approvals: "2"           # require 2 Slack approvals
    keel.sh/approvals-deadline: "24" # hours to approve before it expires
    keel.sh/notify: "#deployments"
```

When a new image is available, Keel posts to `#infra-approvals`:

```
:rocket: Approval needed: myapp:1.3.0 in namespace production
Current: 1.2.0 → New: 1.3.0
React with :white_check_mark: to approve or :x: to reject
```

Two team members react with the checkmark emoji, and Keel proceeds with the rollout. The approval deadline ensures stale approvals do not hang open indefinitely.

**Force-pull latest tag (useful during active development):**

```yaml
annotations:
  keel.sh/policy: force
  keel.sh/trigger: poll
  keel.sh/match-tag: "true"   # only update if tag matches current tag exactly
```

With `force` policy, Keel updates the Deployment even if the tag name does not change (e.g., `latest`). The `match-tag: "true"` annotation restricts this to the exact tag already running, avoiding accidental updates if the registry has multiple tags.

---

## Webhook Trigger from CI/CD

Instead of polling, trigger Keel immediately when your CI pipeline pushes an image:

```yaml
# .github/workflows/deploy.yml
- name: Trigger Keel update
  run: |
    curl -X POST https://keel.yourcluster.internal/v1/webhooks/native \
      -H "Content-Type: application/json" \
      -d '{
        "name": "ghcr.io/yourorg/myapp",
        "tag": "${{ github.sha }}"
      }'
```

This fires a native Keel webhook that immediately checks matching workloads against the new tag.

For multi-environment pipelines, send separate webhook calls per environment with environment-specific tags:

```yaml
- name: Build and push image
  run: |
    docker build -t ghcr.io/yourorg/myapp:${{ github.sha }} .
    docker push ghcr.io/yourorg/myapp:${{ github.sha }}

- name: Trigger staging deploy
  run: |
    curl -X POST https://keel.yourcluster.internal/v1/webhooks/native \
      -H "Content-Type: application/json" \
      -d '{"name": "ghcr.io/yourorg/myapp", "tag": "${{ github.sha }}"}'

- name: Wait for staging verification
  run: sleep 120  # or use a health check poll

- name: Tag and trigger production deploy
  if: github.ref == 'refs/heads/main'
  run: |
    docker tag ghcr.io/yourorg/myapp:${{ github.sha }} \
               ghcr.io/yourorg/myapp:prod-${{ github.sha }}
    docker push ghcr.io/yourorg/myapp:prod-${{ github.sha }}
    curl -X POST https://keel.yourcluster.internal/v1/webhooks/native \
      -H "Content-Type: application/json" \
      -d '{"name": "ghcr.io/yourorg/myapp", "tag": "prod-${{ github.sha }}"}'
```

---

## Registry Webhooks for GHCR and Docker Hub

**GitHub Container Registry (GHCR):**

Configure a GitHub webhook on push events pointing to:

```
https://keel.yourcluster.internal/v1/webhooks/github
```

```bash
gh api repos/yourorg/yourrepo/hooks --method POST \
  -f "config[url]=https://keel.yourcluster.internal/v1/webhooks/github" \
  -f "config[content_type]=json" \
  -f events[]="push" \
  -f active=true
```

**Docker Hub:**

In Docker Hub → Repository → Webhooks, add:

```
https://keel.yourcluster.internal/v1/webhooks/dockerhub
```

**Amazon ECR:**

ECR does not support webhooks natively. Use EventBridge + Lambda to forward ECR push events to Keel:

```bash
# Lambda function that calls the Keel native webhook
import json, urllib.request

def handler(event, context):
    detail = event['detail']
    image = f"{detail['repository-name']}:{detail['image-tag']}"
    payload = json.dumps({"name": detail['repository-name'], "tag": detail['image-tag']})
    req = urllib.request.Request(
        "https://keel.yourcluster.internal/v1/webhooks/native",
        data=payload.encode(),
        headers={"Content-Type": "application/json"},
        method="POST"
    )
    urllib.request.urlopen(req)
```

---

## Update Helm Releases with Keel

Keel can also update Helm release values when a new image is published. Add a `keel` section to your Helm values:

```yaml
# values.yaml for a Helm chart managed by Keel
image:
  repository: ghcr.io/yourorg/myapp
  tag: "1.2.0"

keel:
  policy: patch          # only auto-update patch versions
  trigger: poll
  images:
    - repository: image.repository
      tag: image.tag
  approvals: 0
  notify:
    slack:
      channel: "#deployments"
```

Keel patches the Helm release values directly and triggers a `helm upgrade`.

For Helm charts with multiple containers, specify each image path:

```yaml
keel:
  policy: minor
  trigger: poll
  images:
    - repository: app.image.repository
      tag: app.image.tag
    - repository: sidecar.image.repository
      tag: sidecar.image.tag
  approvals: 1
```

---

## Monitoring Keel

```bash
# Check Keel logs
kubectl logs -n keel -l app=keel -f

# List pending approvals
kubectl exec -n keel deploy/keel -- keel approvals --pending

# List all tracked images
kubectl exec -n keel deploy/keel -- keel tracked

# Force a manual update check
kubectl exec -n keel deploy/keel -- keel update --name myapp --namespace staging
```

**Prometheus metrics** are exposed at `:9300/metrics`. Key metrics:

```
keel_update_approval_total        — total approvals sent
keel_update_deployment_total      — total deployments triggered
keel_registry_pull_errors_total   — registry pull failures
```

Add a Grafana dashboard using these metrics to track deployment frequency and approval rate over time. A rising `keel_registry_pull_errors_total` is an early warning of registry authentication failures or rate limiting.

For alerting on stuck approvals, create a Prometheus rule:

```yaml
groups:
  - name: keel
    rules:
      - alert: KeelApprovalPending
        expr: increase(keel_update_approval_total[2h]) > 0
          and increase(keel_update_deployment_total[2h]) == 0
        for: 2h
        labels:
          severity: warning
        annotations:
          summary: "Keel has a pending approval that has not been acted on"
```

---

## Common Issues and Fixes

**Keel is not picking up a new image tag**

Check that the Deployment annotation `keel.sh/policy` is set and that the image path in the Deployment spec exactly matches the registry path Keel received in the webhook. Tags are case-sensitive.

```bash
# Confirm Keel sees the workload
kubectl exec -n keel deploy/keel -- keel tracked | grep myapp
```

**Slack approval emoji not being detected**

Keel reads emoji reactions, not message replies. The bot must be a member of the `#infra-approvals` channel. Use `/invite @keel` in Slack if Keel is not already in the channel.

**Keel updated a Deployment I did not want auto-updated**

Remove or set `keel.sh/policy: ""` on the Deployment. Keel only manages workloads that explicitly opt in via annotation. If a workload was added without an annotation and Keel still updated it, check if it is part of a Helm release that has Keel values configured.

**Image pull backoff after Keel update**

The new tag does not exist in the registry or the pull credentials are wrong. Check `imagePullSecrets` on the pod spec and confirm the tag was actually pushed before the webhook fired. Adding a `sleep 10` between the push step and the webhook call in your CI pipeline is usually enough buffer.

**Compacted poll interval causing delayed staging updates**

If polling is set to `@every 3m` but your CI publishes multiple images per hour, switch to webhook triggers to get sub-second propagation.

---

## Keel vs. ArgoCD: When to Use Each

| Concern | Keel | ArgoCD |
|---------|------|--------|
| Source of truth | Image registry | Git repository |
| Best for | Staging auto-deploy, fast iteration | Production GitOps with auditability |
| Config changes | No (image tags only) | Yes (all manifests) |
| Rollback | Manual (re-tag or annotate) | Git revert → auto-sync |
| Complexity | Low | Higher |

Run both: Keel for staging instant-deploy, ArgoCD for production GitOps. Use Keel's webhook trigger at the end of the CI pipeline to update staging immediately, and let ArgoCD manage production through pull requests merged to a GitOps repository.

---

## Related Reading

- [How to Set Up ArgoCD for GitOps Workflows](/remote-work-tools/argocd-gitops-workflow-setup/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)
- [How to Create Automated Rollback Systems](/remote-work-tools/automated-rollback-systems/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
