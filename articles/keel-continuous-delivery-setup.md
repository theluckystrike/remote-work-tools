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

---

## How Keel Works

Keel runs as a Deployment in your cluster. It monitors container registries (by polling or webhook), compares image tags against your running workloads, and updates them according to per-workload update policies defined in annotations or Helm values.

Update policies:
- `all`: update on any new tag
- `major`, `minor`, `patch`: semver-aware updates
- `force`: force-pull the same tag (useful for `latest`)
- `glob:prod-*`: update when a tag matching a glob is pushed

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

---

## Keel vs. ArgoCD: When to Use Each

| Concern | Keel | ArgoCD |
|---------|------|--------|
| Source of truth | Image registry | Git repository |
| Best for | Staging auto-deploy, fast iteration | Production GitOps with auditability |
| Config changes | No (image tags only) | Yes (all manifests) |
| Rollback | Manual (re-tag or annotate) | Git revert → auto-sync |
| Complexity | Low | Higher |

Run both: Keel for staging instant-deploy, ArgoCD for production GitOps.

---

## Related Reading

- [How to Set Up ArgoCD for GitOps Workflows](/remote-work-tools/argocd-gitops-workflow-setup/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)
- [How to Create Automated Rollback Systems](/remote-work-tools/automated-rollback-systems/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
