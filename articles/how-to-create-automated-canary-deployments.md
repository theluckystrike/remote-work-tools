---
layout: default
title: "How to Create Automated Canary Deployments"
description: "Implement automated canary deployments with Argo Rollouts, Flagger, or Nginx weighted routing to reduce deployment risk for remote teams"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-automated-canary-deployments/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Canary deployments route a small percentage of traffic to a new version while the rest continues on the stable version. If the canary shows elevated errors or latency, traffic is automatically rolled back. For remote teams deploying across time zones, automated promotion means you can ship at any hour without someone watching a dashboard.

---

## Option 1: Argo Rollouts (Kubernetes)

Argo Rollouts extends Kubernetes deployments with progressive delivery strategies.

Install:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install CLI
brew install argoproj/tap/kubectl-argo-rollouts
```

Define a canary rollout:

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payments-service
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: payments-service
  template:
    metadata:
      labels:
        app: payments-service
    spec:
      containers:
        - name: payments-service
          image: yourcompany/payments-service:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
  strategy:
    canary:
      # Step-by-step traffic progression
      steps:
        - setWeight: 5     # 5% to canary
        - pause:
            duration: 5m   # Wait 5 minutes, check metrics
        - setWeight: 20
        - pause:
            duration: 5m
        - setWeight: 50
        - pause:
            duration: 10m
        - setWeight: 80
        - pause:
            duration: 5m
        # Final step: 100% — rollout complete

      # Automatic analysis at each step
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
        args:
          - name: service-name
            value: payments-service

      canaryService: payments-service-canary
      stableService: payments-service-stable
```

Define the analysis template to check error rate:

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      # Fail if success rate drops below 95%
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            sum(rate(http_requests_total{
              service="{{ args.service-name }}",
              status!~"5.."
            }[5m])) /
            sum(rate(http_requests_total{
              service="{{ args.service-name }}"
            }[5m]))

    - name: latency-p99
      interval: 1m
      successCondition: result[0] < 0.5
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring.svc:9090
          query: |
            histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{
              service="{{ args.service-name }}"
            }[5m]))
```

Deploy and monitor:

```bash
kubectl apply -f rollout.yaml -f analysis-template.yaml

# Watch rollout progress
kubectl argo rollouts get rollout payments-service --watch

# Manually promote if paused
kubectl argo rollouts promote payments-service

# Abort if something looks wrong
kubectl argo rollouts abort payments-service
```

---

## Option 2: Flagger (Kubernetes + Service Mesh)

Flagger automates canary analysis using Istio, Linkerd, or Nginx ingress for traffic splitting.

Install Flagger with Nginx:

```bash
helm repo add flagger https://flagger.app
helm upgrade -i flagger flagger/flagger \
  --namespace ingress-nginx \
  --set meshProvider=nginx \
  --set metricsServer=http://prometheus.monitoring:9090
```

Create a Canary resource:

```yaml
# canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payments-service
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payments-service
  ingress:
    name: payments-service
  progressDeadlineSeconds: 120
  service:
    port: 80
    targetPort: 8080
    gateways:
      - public-gateway.istio-system.svc.cluster.local
    hosts:
      - payments.yourcompany.com
  analysis:
    interval: 1m
    threshold: 5       # Max failed metric checks before rollback
    maxWeight: 50      # Max % to canary
    stepWeight: 10     # Increment per step
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://payments-service-canary/api/health | grep ok"
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 http://payments-service-canary/"
```

Trigger a canary by updating the deployment image:

```bash
kubectl set image deployment/payments-service \
  payments-service=yourcompany/payments-service:v2.0.1 \
  -n production

# Flagger automatically detects the change and starts the canary
kubectl describe canary payments-service -n production
```

---

## Option 3: Nginx Weighted Routing (Simple)

No Kubernetes? Use Nginx with a split_clients module for simple canary routing:

```nginx
# /etc/nginx/conf.d/canary.conf
# Route 10% to canary, 90% to stable

split_clients "${remote_addr}${http_user_agent}${msec}" $canary_upstream {
    10%   canary;
    *     stable;
}

upstream stable {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

upstream canary {
    server 10.0.2.10:8080;
}

server {
    listen 443 ssl http2;
    server_name api.yourcompany.com;

    location / {
        proxy_pass http://$canary_upstream;
        proxy_set_header Host $host;
        proxy_set_header X-Canary $canary_upstream;
        add_header X-Served-By $canary_upstream;
    }
}
```

Reload Nginx to apply:

```bash
nginx -t && systemctl reload nginx
```

To adjust the percentage, change `10%` and reload. To complete the rollout, change to `100%`. To rollback, change to `0%`.

---

## Automated Rollback with a Monitoring Script

Without Kubernetes, monitor the canary manually with a script:

```bash
#!/bin/bash
# scripts/canary-monitor.sh
# Monitor canary error rate and auto-rollback if threshold exceeded

CANARY_URL="https://api.yourcompany.com"
ERROR_THRESHOLD=5    # Percentage
CHECK_DURATION=300   # 5 minutes
CHECK_INTERVAL=30    # Check every 30 seconds

total_requests=0
error_requests=0
end_time=$(($(date +%s) + CHECK_DURATION))

echo "Monitoring canary for $CHECK_DURATION seconds..."

while [ $(date +%s) -lt $end_time ]; do
  # Sample error rate from metrics endpoint
  stats=$(curl -sf "$CANARY_URL/internal/metrics" | jq '{total: .requests_total, errors: .errors_total}')
  period_total=$(echo "$stats" | jq '.total')
  period_errors=$(echo "$stats" | jq '.errors')

  if [ "$period_total" -gt 0 ]; then
    error_rate=$(echo "scale=2; $period_errors * 100 / $period_total" | bc)
    echo "$(date): Error rate: ${error_rate}% (${period_errors}/${period_total})"

    if (( $(echo "$error_rate > $ERROR_THRESHOLD" | bc -l) )); then
      echo "ERROR: Canary error rate ${error_rate}% exceeds threshold ${ERROR_THRESHOLD}%. Rolling back."

      # Update Nginx config to route 0% to canary
      sed -i 's/10%/0%/' /etc/nginx/conf.d/canary.conf
      nginx -t && systemctl reload nginx

      curl -s -X POST \
        -H 'Content-type: application/json' \
        --data "{\"text\":\"CANARY ROLLBACK: error rate ${error_rate}% exceeded ${ERROR_THRESHOLD}%\"}" \
        "$SLACK_WEBHOOK"
      exit 1
    fi
  fi

  sleep $CHECK_INTERVAL
done

echo "Canary passed monitoring period. Proceeding with full rollout."
```

---

## Related Reading

- [Best Tools for Remote Team Feature Flags](/remote-work-tools/best-tools-remote-team-feature-flags/)
- [How to Set Up Drone CI for Remote Teams](/remote-work-tools/how-to-set-up-drone-ci-for-remote-teams/)
- [How to Create Automated Status Pages](/remote-work-tools/how-to-create-automated-status-pages/)
- [How to Create Automated Client Progress Report for Remote](/remote-work-tools/how-to-create-automated-client-progress-report-for-remote-pr/)

---

## Related Articles

- [How to Set Up Canary Tokens for Detecting Unauthorized](/remote-work-tools/how-to-set-up-canary-tokens-for-detecting-unauthorized-acces/)
- [How to Create Automated Status Pages](/remote-work-tools/how-to-create-automated-status-pages/)
- [How to Create Remote Team Style Guides](/remote-work-tools/how-to-create-remote-team-style-guides/)
- [How to Create Automated Client Progress Report for Remote](/remote-work-tools/how-to-create-automated-client-progress-report-for-remote-pr/)
- [How to Create a Remote Dev Environment Template](/remote-work-tools/how-to-create-a-remote-dev-environment-template/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
