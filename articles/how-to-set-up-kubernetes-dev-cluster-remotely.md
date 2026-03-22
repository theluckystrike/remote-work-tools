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

## Table of Contents

- [Why k3s Over Full Kubernetes](#why-k3s-over-full-kubernetes)
- [Server Requirements](#server-requirements)
- [Prerequisites](#prerequisites)
- [Debugging Common Issues](#debugging-common-issues)
- [Troubleshooting](#troubleshooting)
- [Related Reading](#related-reading)

## Why k3s Over Full Kubernetes

k3s uses under 512MB RAM at idle, installs in 30 seconds, and handles everything a remote dev team needs. It runs containerd, CoreDNS, Traefik ingress, and local storage provisioner out of the box.

Full Kubernetes (kubeadm-based) requires significantly more overhead: a dedicated etcd cluster, manual CNI installation, and node configuration scripts that take 20-30 minutes to stabilize. For a shared dev environment, that complexity adds friction without meaningful benefit. k3s also packages SQLite as an embedded datastore for single-node setups, making backup and restore trivial.

## Server Requirements

- Ubuntu 22.04 LTS (2 vCPU, 4GB RAM minimum per node)
- Open ports: 6443 (API), 80, 443 (ingress), 8472/udp (Flannel VXLAN)

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: Install k3s Server Node

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

### Step 2: Add Worker Nodes

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

### Step 3: Kubeconfig for Team Access

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

### Step 4: Namespace-Based Team Isolation

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

### Step 5: Install Helm

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

### Step 6: Deploy PostgreSQL with Helm

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

### Step 7: Skaffold for Fast Iteration

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

### Step 8: Traefik Ingress Configuration

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

### Step 9: Resource Quotas

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

### Step 10: Persistent Storage with Longhorn

For dev clusters that need reliable persistent volumes across node restarts, Longhorn provides replicated block storage without the complexity of Ceph:

```bash
# Install Longhorn via Helm
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --set defaultSettings.defaultReplicaCount=2

# Verify Longhorn pods are running
kubectl -n longhorn-system get pods

# Set Longhorn as the default storage class
kubectl patch storageclass longhorn \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Once Longhorn is running, PersistentVolumeClaims automatically get distributed storage. Your Helm deployments that specify `storageClassName: longhorn` (or no class, since it's default) will get volumes that survive node failures and can be snapshotted for backup.

### Step 11: Cluster Autoscaling for Cost Control

Dev clusters on cloud VMs can burn budget fast. Use a simple cron-based scale-down during off-hours rather than full cluster autoscaler complexity:

```bash
# Scale down all deployments in dev namespaces at 10 PM
# Store replica counts as annotations before scaling
kubectl get deployments -n dev-alice -o json | jq -r \
  '.items[] | "\(.metadata.name) \(.spec.replicas)"' | \
  while read name replicas; do
    kubectl annotate deployment/$name -n dev-alice \
      saved-replicas=$replicas --overwrite
    kubectl scale deployment/$name -n dev-alice --replicas=0
  done

# Restore in the morning
kubectl get deployments -n dev-alice -o json | jq -r \
  '.items[] | "\(.metadata.name) \(.metadata.annotations["saved-replicas"] // "1")"' | \
  while read name replicas; do
    kubectl scale deployment/$name -n dev-alice --replicas=$replicas
  done
```

Wrap these scripts in a Kubernetes CronJob using the `bitnami/kubectl` image and mount the right ServiceAccount, and you get automatic cost savings with zero manual intervention.

## Debugging Common Issues

### Pods stuck in Pending

The most frequent cause in a resource-constrained dev cluster is insufficient CPU or memory:

```bash
kubectl describe pod <pod-name> -n dev-alice
# Look for: "0/2 nodes are available: 2 Insufficient memory"

# Check node resources
kubectl top nodes
kubectl describe node dev-worker1 | grep -A5 "Allocated resources"
```

Either reduce resource requests in the Helm values or apply a node with more capacity.

### ImagePullBackOff

Private registries need a pull secret in each namespace:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=your-registry.example.com \
  --docker-username=robot \
  --docker-password=your-token \
  --namespace dev-alice

# Reference in deployment
# spec.template.spec.imagePullSecrets:
#   - name: regcred
```

### Step 12: Shared Container Registry Access

Remote team members need a registry that all developers and the cluster can pull from. Self-hosted Harbor is the most capable option, but for smaller teams a cloud registry with a shared robot account works fine.

For team setups, create per-namespace pull secrets from a single registry robot account, then patch the default ServiceAccount to always use it:

```bash
# Create pull secret in every dev namespace
for ns in dev-alice dev-bob staging; do
  kubectl create secret docker-registry regcred \
    --docker-server=registry.example.com \
    --docker-username=robot \
    --docker-password=$(cat /path/to/robot-token) \
    --namespace $ns

  # Patch default service account so all pods auto-use it
  kubectl patch serviceaccount default \
    -n $ns \
    -p '{"imagePullSecrets": [{"name": "regcred"}]}'
done
```

With this in place, every pod in those namespaces automatically pulls from the private registry without requiring `imagePullSecrets` in each manifest.

### Step 13: Set Up Metrics Server for HPA

Horizontal Pod Autoscaler requires the metrics-server to be running. k3s ships without it by default:

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# k3s may need the --kubelet-insecure-tls flag due to self-signed certs
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify it works
kubectl top nodes
kubectl top pods -n dev-alice
```

Once metrics-server is running, you can configure HPAs on any deployment:

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
  namespace: dev-alice
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Step 14: Monitor with k9s

```bash
# Install k9s for terminal cluster management
brew install k9s  # macOS
# or
curl -sS https://webinstall.dev/k9s | bash  # Linux

k9s --namespace dev-alice
# Navigate: :pods, :services, :logs, :exec
```

### Step 15: Upgrading k3s

k3s upgrades are non-disruptive when done node-by-node. The upgrade controller handles this automatically:

```bash
# Install the k3s upgrade controller
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/latest/download/system-upgrade-controller.yaml

# Define an upgrade plan
cat <<EOF | kubectl apply -f -
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/control-plane, operator: Exists}
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  channel: https://update.k3s.io/v1-release/channels/stable
EOF
```

The controller drains nodes, applies the upgrade, and uncordons them. Worker nodes get upgraded after the control plane is done, ensuring zero downtime for running workloads.

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Related Reading

- [How to Secure Remote Team Kubernetes Clusters](/remote-work-tools/how-to-secure-remote-team-kubernetes-clusters-with-network-p/)
- [Best Container Registry Tool for Remote Teams](/remote-work-tools/best-container-registry-tool-for-remote-teams-sharing-docker/)
- [Setting Up Harbor for Container Registry](/remote-work-tools/setting-up-harbor-for-container-registry/)
- [How to Set Up a Soundproof Home Office When Working](/remote-work-tools/how-to-set-up-soundproof-home-office-when-working-remotely-w/)

---

## Related Articles

- [Setting Up a Remote Dev Server with Hetzner](/remote-work-tools/setting-up-remote-dev-server-with-hetzner/)
- [How to Secure Remote Team Kubernetes Clusters with Network P](/remote-work-tools/how-to-secure-remote-team-kubernetes-clusters-with-network-p/)
- [Portable Dev Environment with Docker 2026](/remote-work-tools/portable-dev-environment-docker-2026/)
- [How to Create a Remote Dev Environment Template](/remote-work-tools/how-to-create-a-remote-dev-environment-template/)
- [How to Automate Dev Environment Setup: A Practical Guide](/remote-work-tools/how-to-automate-dev-environment-setup/)
Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
