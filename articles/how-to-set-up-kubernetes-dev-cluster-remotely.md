---
layout: default
title: "How to Set Up a Kubernetes Dev Cluster Remotely"
description: "Spin up a remote Kubernetes dev cluster with k3s, configure kubeconfig for team access, and deploy apps with Helm and Skaffold"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-kubernetes-dev-cluster-remotely/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Running a shared Kubernetes dev cluster lets remote teams test against a real cluster without local resource constraints. This guide uses k3s for lightweight deployment, Helm for app management, and kubeconfig sharing patterns for distributed teams.

## Why k3s Over Full Kubernetes

k3s uses under 512MB RAM at idle, installs in 30 seconds, and handles everything a remote dev team needs. It runs containerd, CoreDNS, Traefik ingress, and local storage provisioner out of the box.

## Server Requirements

- Ubuntu 22.04 LTS (2 vCPU, 4GB RAM minimum per node)
- Open ports: 6443 (API), 80, 443 (ingress), 8472/udp (Flannel VXLAN)

## Install k3s Server Node

```bash
# Install k3s with Traefik ingress and no local storage (use Longhorn instead)
curl -sfL https://get.k3s.io | sh -s - server \
  --tls-san your-cluster.example.com \
  --tls-san $(curl -s ifconfig.me) \
  --disable local-storage \
  --write-kubeconfig-mode 644 \
  --cluster-init

# Verify installation
sudo k3s kubectl get nodes
# NAME         STATUS   ROLES                  AGE   VERSION
# dev-master   Ready    control-plane,master   60s   v1.28.x+k3s1

# Get node token for workers
sudo cat /var/lib/rancher/k3s/server/node-token
```

## Add Worker Nodes

```bash
# On each worker node:
K3S_TOKEN="your-node-token-here"
K3S_URL="https://your-cluster.example.com:6443"

curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN K3S_URL=$K3S_URL sh -s - agent

# Verify from master:
sudo k3s kubectl get nodes
# NAME         STATUS   ROLES                  AGE
# dev-master   Ready    control-plane,master   5m
# dev-worker1  Ready    <none>                 2m
# dev-worker2  Ready    <none>                 1m
```

## Kubeconfig for Team Access

```bash
# Export kubeconfig from server
sudo cat /etc/rancher/k3s/k3s.yaml

# Replace localhost with the public IP/hostname
sudo sed 's/127.0.0.1/your-cluster.example.com/g' /etc/rancher/k3s/k3s.yaml > ~/team-kubeconfig.yaml

# On team member machines:
mkdir -p ~/.kube
scp deploy@your-cluster.example.com:~/team-kubeconfig.yaml ~/.kube/dev-cluster.yaml

# Use specific config
export KUBECONFIG=~/.kube/dev-cluster.yaml
kubectl get nodes

# Merge with existing config
KUBECONFIG=~/.kube/config:~/.kube/dev-cluster.yaml kubectl config view --merge --flatten > ~/.kube/merged.yaml
mv ~/.kube/merged.yaml ~/.kube/config
kubectl config use-context default
```

## Namespace-Based Team Isolation

Give each developer or team their own namespace with RBAC:

```yaml
# namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev-alice
  labels:
    team: engineering
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev-bob
  labels:
    team: engineering
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

```yaml
# rbac-developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev-alice
  name: developer
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: dev-alice
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f namespaces.yaml
kubectl apply -f rbac-developer.yaml

# Set default namespace for a developer
kubectl config set-context --current --namespace=dev-alice
```

## Install Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add common repos
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# List available charts
helm search repo bitnami/postgres
```

## Deploy PostgreSQL with Helm

```bash
helm install postgres bitnami/postgresql \
  --namespace dev-alice \
  --set auth.postgresPassword=devpassword \
  --set primary.persistence.size=2Gi \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.requests.cpu=100m

# Connect to database
kubectl run psql-client --rm --tty -i --restart='Never' \
  --namespace dev-alice \
  --image docker.io/bitnami/postgresql:15 \
  --env="PGPASSWORD=devpassword" \
  --command -- psql --host postgres-postgresql --username postgres --port 5432
```

## Skaffold for Fast Iteration

Skaffold handles build-push-deploy in a single command:

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: my-app

build:
  local:
    push: true
  artifacts:
    - image: your-registry.example.com/my-app
      docker:
        dockerfile: Dockerfile
      sync:
        manual:
          - src: "src/**/*.py"
            dest: /app

deploy:
  helm:
    releases:
      - name: my-app
        chartPath: ./helm/my-app
        namespace: dev-alice
        setValues:
          image.repository: your-registry.example.com/my-app
          image.tag: "@sha256"
          replicaCount: 1

portForward:
  - resourceType: service
    resourceName: my-app
    namespace: dev-alice
    port: 8080
    localPort: 8080
```

```bash
# Develop with live reload
skaffold dev --namespace=dev-alice

# Deploy once
skaffold run --namespace=dev-alice

# Clean up
skaffold delete --namespace=dev-alice
```

## Traefik Ingress Configuration

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: dev-alice
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
spec:
  rules:
    - host: alice.dev.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 8080
  tls:
    - hosts:
        - alice.dev.example.com
```

## Resource Quotas

Prevent any one namespace from consuming all cluster resources:

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev-alice
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
```

```bash
kubectl apply -f resource-quota.yaml
kubectl describe resourcequota dev-quota -n dev-alice
```

## Monitoring with k9s

```bash
# Install k9s for terminal cluster management
brew install k9s  # macOS
# or
curl -sS https://webinstall.dev/k9s | bash  # Linux

k9s --namespace dev-alice
# Navigate: :pods, :services, :logs, :exec
```

## Related Reading

- [How to Secure Remote Team Kubernetes Clusters](/remote-work-tools/how-to-secure-remote-team-kubernetes-clusters-with-network-p/)
- [Best Container Registry Tool for Remote Teams](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Setting Up Harbor for Container Registry](/remote-work-tools/setting-up-harbor-for-container-registry/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
