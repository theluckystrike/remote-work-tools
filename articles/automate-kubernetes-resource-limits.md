---
layout: default
title: "How to Automate Kubernetes Resource Limits"
description: "Automatically set and right-size Kubernetes CPU and memory limits using VPA, Goldilocks, LimitRange policies, and custom scripts that prevent OOMKills and throttling"
date: 2026-03-22
author: theluckystrike
permalink: /automate-kubernetes-resource-limits/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Automate Kubernetes Resource Limits

Setting Kubernetes resource limits manually means guessing, and guessing wrong in either direction hurts: too low causes OOMKills and CPU throttling; too high wastes money and starves other pods. Automation shifts this from a one-time guess to a continuous process that adjusts limits based on actual usage.

---

## Why Manual Limits Fail

A developer writes `requests: memory: 256Mi, cpu: 100m` because those are round numbers. Three months later the service handles 10x the traffic, gets OOMKilled weekly, and nobody knows why because the resource requests haven't changed. The answer is to measure real usage and automate the limits from that data.

---

## Approach 1: Vertical Pod Autoscaler (VPA) — Recommendation Mode

VPA watches pod resource usage over time and generates recommendations. In `Off` mode it only recommends — you can review and apply changes in your own process. In `Auto` mode it updates the pod spec and restarts pods.

**Install VPA:**

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

**Create a VPA object in Off mode (recommendations only):**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"   # recommendations only, no automatic restarts
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 4
          memory: 4Gi
        controlledResources: ["cpu", "memory"]
```

**Read recommendations:**

```bash
kubectl get vpa myapp-vpa -n production -o json | \
  jq '.status.recommendation.containerRecommendations[0]'
```

Output:

```json
{
  "containerName": "myapp",
  "lowerBound": { "cpu": "88m",   "memory": "128Mi" },
  "target":      { "cpu": "180m", "memory": "256Mi" },
  "upperBound":  { "cpu": "420m", "memory": "512Mi" }
}
```

**Script to apply VPA recommendations as actual limits:**

```bash
#!/bin/bash
# apply-vpa-recommendations.sh
# Reads VPA target recommendations and patches deployment resource limits

NAMESPACE="${1:-production}"
DRY_RUN="${2:-true}"  # pass "false" to apply

for vpa in $(kubectl get vpa -n "$NAMESPACE" -o jsonpath='{.items[*].metadata.name}'); do
  deployment=$(kubectl get vpa "$vpa" -n "$NAMESPACE" \
    -o jsonpath='{.spec.targetRef.name}')

  recommendations=$(kubectl get vpa "$vpa" -n "$NAMESPACE" -o json \
    | jq -c '.status.recommendation.containerRecommendations[]')

  echo "=== $vpa → $deployment ==="
  echo "$recommendations" | while IFS= read -r rec; do
    container=$(echo "$rec" | jq -r '.containerName')
    cpu=$(echo "$rec"    | jq -r '.target.cpu')
    memory=$(echo "$rec" | jq -r '.target.memory')

    echo "  $container: cpu=$cpu memory=$memory"

    if [[ "$DRY_RUN" == "false" ]]; then
      kubectl set resources deployment/"$deployment" \
        -n "$NAMESPACE" \
        -c "$container" \
        --requests="cpu=${cpu},memory=${memory}" \
        --limits="cpu=$(echo "$rec" | jq -r '.upperBound.cpu'),memory=$(echo "$rec" | jq -r '.upperBound.memory')"
    fi
  done
done

[[ "$DRY_RUN" == "true" ]] && echo "(dry run — pass 'false' as second arg to apply)"
```

---

## Approach 2: Goldilocks Dashboard

Goldilocks runs VPA in recommendation mode for every deployment in a namespace and provides a web UI showing current vs. recommended limits.

```bash
# Install via Helm
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm repo update

helm install goldilocks fairwinds-stable/goldilocks \
  --namespace goldilocks \
  --create-namespace \
  --set controller.flags.on-by-default=false

# Enable Goldilocks for a specific namespace
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Access the dashboard
kubectl port-forward -n goldilocks svc/goldilocks-dashboard 8080:80
# Open http://localhost:8080
```

Goldilocks shows a table per deployment with current requests/limits and VPA recommendations in a copy-paste format ready for your Helm values.

---

## Approach 3: LimitRange — Enforce Defaults

LimitRange automatically injects default resource requests and limits into any pod that doesn't specify them. It prevents unbounded pods from consuming all cluster resources.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:        # applied as limits if not specified
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: 4
        memory: 4Gi
    - type: Pod
      max:
        cpu: 8
        memory: 8Gi
```

```bash
kubectl apply -f limitrange.yaml -n production

# Verify it's active
kubectl describe limitrange default-limits -n production
```

---

## Approach 4: ResourceQuota — Namespace-Level Budget

Set a hard ceiling on total resource consumption per namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    persistentvolumeclaims: "20"
```

```bash
# Check current usage against quota
kubectl get resourcequota -n production
kubectl describe resourcequota production-quota -n production
```

---

## Automated Weekly Right-Sizing Report

Run weekly and post to Slack when pods are significantly over-provisioned:

```python
#!/usr/bin/env python3
# right-size-report.py
import subprocess
import json
import os
import requests

NAMESPACE = os.environ.get("K8S_NAMESPACE", "production")
SLACK_WEBHOOK = os.environ.get("SLACK_WEBHOOK_URL", "")
OVERPROVISIONED_THRESHOLD = 0.3  # flag if using < 30% of requested CPU

def get_top_pods():
    result = subprocess.run(
        ["kubectl", "top", "pods", "-n", NAMESPACE, "--no-headers"],
        capture_output=True, text=True
    )
    pods = {}
    for line in result.stdout.strip().split("\n"):
        if not line:
            continue
        parts = line.split()
        name = parts[0]
        cpu_str = parts[1].rstrip("m")
        mem_str = parts[2].rstrip("Mi")
        pods[name] = {
            "cpu_actual_m": int(cpu_str) if cpu_str.isdigit() else 0,
            "mem_actual_mi": int(mem_str) if mem_str.isdigit() else 0,
        }
    return pods

def get_pod_requests():
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", NAMESPACE, "-o", "json"],
        capture_output=True, text=True
    )
    data = json.loads(result.stdout)
    requests_map = {}
    for pod in data["items"]:
        name = pod["metadata"]["name"]
        for container in pod["spec"].get("containers", []):
            res = container.get("resources", {}).get("requests", {})
            cpu_req = res.get("cpu", "0m").rstrip("m")
            mem_req = res.get("memory", "0Mi").rstrip("Mi")
            if name not in requests_map:
                requests_map[name] = {"cpu_req_m": 0, "mem_req_mi": 0}
            requests_map[name]["cpu_req_m"] += int(cpu_req) if cpu_req.isdigit() else 0
            requests_map[name]["mem_req_mi"] += int(mem_req) if mem_req.isdigit() else 0
    return requests_map

actual = get_top_pods()
requested = get_pod_requests()

overprovisioned = []
for pod, act in actual.items():
    req = requested.get(pod, {})
    cpu_req = req.get("cpu_req_m", 0)
    if cpu_req > 0 and act["cpu_actual_m"] / cpu_req < OVERPROVISIONED_THRESHOLD:
        ratio = round(act["cpu_actual_m"] / cpu_req, 2)
        overprovisioned.append(
            f"  {pod}: using {act['cpu_actual_m']}m of {cpu_req}m requested ({ratio*100:.0f}%)"
        )

if overprovisioned and SLACK_WEBHOOK:
    message = f":chart_with_downwards_trend: *Over-provisioned pods in {NAMESPACE}*\n" + "\n".join(overprovisioned[:10])
    requests.post(SLACK_WEBHOOK, json={"text": message})
    print(f"Posted {len(overprovisioned)} over-provisioned pods to Slack")
else:
    print("\n".join(overprovisioned) or "No over-provisioned pods found")
```

---

## Related Reading

- [Prometheus Alerting for Remote Infrastructure](/remote-work-tools/prometheus-alerting-remote-infra-setup/)
- [How to Set Up ArgoCD for GitOps Workflows](/remote-work-tools/argocd-gitops-workflow-setup/)
- [How to Set Up Keel for Continuous Delivery](/remote-work-tools/keel-continuous-delivery-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
