---
layout: default
title: "How to Set Up ArgoCD for GitOps Workflows"
description: "Deploy ArgoCD on Kubernetes to sync cluster state from a Git repository — with App of Apps pattern, RBAC, SSO, automated sync policies, and multi-cluster setup"
date: 2026-03-22
author: theluckystrike
permalink: /argocd-gitops-workflow-setup/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Set Up ArgoCD for GitOps Workflows

GitOps means your Git repository is the single source of truth for cluster state. You make changes by committing to Git, not by running `kubectl apply`. ArgoCD watches your repo and applies any diff between Git state and cluster state automatically. For remote teams, this eliminates SSH access for deployments, creates an audit trail in Git history, and lets anyone verify what's deployed by reading the repo.

---

## Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl rollout status deployment argocd-server -n argocd

# Get initial admin password
argocd admin initial-password -n argocd
```

**Expose the UI:**

```bash
# Port-forward for local access
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Or expose via Ingress (recommended for teams)
kubectl apply -f - << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.yourcompany.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
  tls:
    - hosts:
        - argocd.yourcompany.internal
      secretName: argocd-tls
EOF
```

---

## Repository Structure (App of Apps Pattern)

The App of Apps pattern has one "root" ArgoCD Application that manages all other Applications. This makes it easy to add new services — just commit a new Application manifest to the config repo.

```
gitops-config/
├── apps/
│   ├── root-app.yaml           -- the App of Apps
│   ├── myapp-staging.yaml
│   ├── myapp-production.yaml
│   ├── monitoring.yaml
│   └── ingress-nginx.yaml
├── staging/
│   └── myapp/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── hpa.yaml
└── production/
    └── myapp/
        ├── deployment.yaml
        ├── service.yaml
        └── hpa.yaml
```

**`apps/root-app.yaml`** — bootstrap this once with `kubectl apply`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/gitops-config.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f apps/root-app.yaml
```

---

## Application Manifests

**`apps/myapp-staging.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: ops-alerts
spec:
  project: default
  source:
    repoURL: https://github.com/yourorg/gitops-config.git
    targetRevision: main
    path: staging/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true      # delete resources removed from Git
      selfHeal: true   # revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**`apps/myapp-production.yaml`** — manual sync only for production:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/yourorg/gitops-config.git
    targetRevision: main
    path: production/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    # No automated sync — require human approval via UI or CLI
    syncOptions:
      - CreateNamespace=true
```

---

## RBAC Configuration

Control who can sync which applications:

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Developers can sync staging, read production
    p, role:developer, applications, sync, */myapp-staging, allow
    p, role:developer, applications, get, */*, allow

    # SREs can sync everything
    p, role:sre, applications, *, */*, allow
    p, role:sre, clusters, *, *, allow

    # Assign roles to groups (via SSO group claim)
    g, yourorg:engineering, role:developer
    g, yourorg:sre, role:sre
    g, yourorg:admin, role:admin
```

---

## SSO with GitHub

Configure GitHub OAuth for team authentication:

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.yourcompany.internal
  dex.config: |
    connectors:
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $GITHUB_CLIENT_ID
          clientSecret: $GITHUB_CLIENT_SECRET
          orgs:
            - name: yourorg
              teams:
                - engineering
                - sre
```

---

## Deploying a New Version

The GitOps deployment flow:

```bash
# 1. Build and push the new image in CI
IMAGE="ghcr.io/yourorg/myapp:$(git rev-parse --short HEAD)"
docker build -t "$IMAGE" .
docker push "$IMAGE"

# 2. Update the image tag in the GitOps repo
cd gitops-config
sed -i "s|image: ghcr.io/yourorg/myapp:.*|image: $IMAGE|" \
  staging/myapp/deployment.yaml

git add staging/myapp/deployment.yaml
git commit -m "chore: deploy myapp $(git -C ../myapp rev-parse --short HEAD) to staging"
git push origin main

# 3. ArgoCD detects the change and syncs automatically (staging)
# For production, trigger sync manually:
argocd app sync myapp-production --auth-token "$ARGOCD_TOKEN"
argocd app wait myapp-production --health --timeout 300
```

---

## ArgoCD CLI Cheatsheet

```bash
# Install CLI
brew install argocd

# Login
argocd login argocd.yourcompany.internal --sso

# List applications
argocd app list

# Check app health and sync status
argocd app get myapp-staging

# Manually sync
argocd app sync myapp-production

# View diff between Git and cluster
argocd app diff myapp-production

# Roll back to previous version
argocd app rollback myapp-production

# Force sync with pruning
argocd app sync myapp-staging --prune --force
```

---

## Related Reading

- [How to Set Up Keel for Continuous Delivery](/remote-work-tools/keel-continuous-delivery-setup/)
- [How to Create Automated Rollback Systems](/remote-work-tools/automated-rollback-systems/)
- [How to Automate Kubernetes Resource Limits](/remote-work-tools/automate-kubernetes-resource-limits/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
