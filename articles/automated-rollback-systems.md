---
layout: default
title: "How to Create Automated Rollback Systems"
description: "Build automated rollback systems for Kubernetes, Docker Compose, and Lambda deployments using health checks, Prometheus metrics, and GitHub Actions failure gates"
date: 2026-03-22
author: theluckystrike
permalink: /automated-rollback-systems/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}
## How to Create Automated Rollback Systems

A deployment that breaks at 3 AM and waits for an on-call engineer to wake up and roll back is a deployment with two failure windows: the failure itself, and the time-to-rollback. Automated rollback closes the second window by detecting the failure and reverting without human intervention.

---

## Kubernetes: Built-In Rollback with Health Gates

Kubernetes has rollback built in via `kubectl rollout undo`. The automation goal is to trigger it based on health conditions, not manual observation.

**Deployment with proper health checks:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # never kill old pods before new ones are ready
      maxSurge: 1          # spin up one extra pod during update
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
        - name: myapp
          image: ghcr.io/yourorg/myapp:latest
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
```

**Automated rollback after a failed rollout:**

```bash
#!/bin/bash
# k8s-deploy-with-rollback.sh
set -euo pipefail

DEPLOYMENT="${1:?Usage: $0 <deployment> <image>}"
IMAGE="${2:?}"
NAMESPACE="${3:-production}"
TIMEOUT="${4:-300}"

echo "Deploying $IMAGE to $DEPLOYMENT in $NAMESPACE"

# Record current image for rollback reference
CURRENT_IMAGE=$(kubectl get deployment "$DEPLOYMENT" \
  -n "$NAMESPACE" \
  -o jsonpath='{.spec.template.spec.containers[0].image}')

# Apply new image
kubectl set image deployment/"$DEPLOYMENT" \
  "${DEPLOYMENT}=${IMAGE}" \
  -n "$NAMESPACE"

# Wait for rollout with timeout
if ! kubectl rollout status deployment/"$DEPLOYMENT" \
  -n "$NAMESPACE" \
  --timeout="${TIMEOUT}s"; then

  echo "ERROR: Rollout failed — rolling back to $CURRENT_IMAGE"

  kubectl rollout undo deployment/"$DEPLOYMENT" -n "$NAMESPACE"
  kubectl rollout status deployment/"$DEPLOYMENT" -n "$NAMESPACE" --timeout=120s

  # Notify Slack
  curl -s -X POST "$SLACK_WEBHOOK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"text\": \":rotating_light: Deployment FAILED and ROLLED BACK\nDeployment: \`$DEPLOYMENT\`\nFailed image: \`$IMAGE\`\nReverted to: \`$CURRENT_IMAGE\`\"}"

  exit 1
fi

echo "Deployment successful: $IMAGE"
```

---

## Kubernetes: Prometheus-Based Rollback

Roll back based on error rate crossing a threshold after deploy — more reliable than timeouts alone:

```bash
#!/bin/bash
# prometheus-gate.sh — post-deploy error rate check
set -euo pipefail

PROM_URL="${PROMETHEUS_URL:-http://prometheus:9090}"
DEPLOYMENT="$1"
THRESHOLD="0.05"  # 5% error rate
OBSERVATION_WINDOW="5m"
WAIT_SECONDS=120   # wait 2 min after deploy before checking

echo "Waiting ${WAIT_SECONDS}s for metrics to stabilize..."
sleep "$WAIT_SECONDS"

# Query 5-minute error rate for this deployment
QUERY="sum(rate(http_requests_total{deployment=\"${DEPLOYMENT}\",status=~\"5..\"}[${OBSERVATION_WINDOW}])) / sum(rate(http_requests_total{deployment=\"${DEPLOYMENT}\"}[${OBSERVATION_WINDOW}]))"

ERROR_RATE=$(curl -s "${PROM_URL}/api/v1/query" \
  --data-urlencode "query=${QUERY}" \
  | jq -r '.data.result[0].value[1] // "0"')

echo "Current error rate: $ERROR_RATE (threshold: $THRESHOLD)"

if (( $(echo "$ERROR_RATE > $THRESHOLD" | bc -l) )); then
  echo "ERROR RATE ABOVE THRESHOLD — triggering rollback"
  kubectl rollout undo deployment/"$DEPLOYMENT" -n production
  kubectl rollout status deployment/"$DEPLOYMENT" -n production --timeout=120s
  exit 1
fi

echo "Error rate OK — deployment accepted"
```

---

## Docker Compose: Health-Check Rollback

For non-Kubernetes deployments with Docker Compose:

```bash
#!/bin/bash
# compose-deploy.sh
set -euo pipefail

COMPOSE_FILE="${1:-/opt/myapp/docker-compose.yml}"
SERVICE="${2:?Usage: $0 <compose-file> <service>}"
HEALTH_URL="${3:?}"  # e.g., http://localhost:3000/health
MAX_WAIT=120

# Save current image tag
CURRENT_IMAGE=$(docker-compose -f "$COMPOSE_FILE" config | \
  awk "/^  ${SERVICE}:/{found=1} found && /image:/{print \$2; exit}")

echo "Pulling new image for $SERVICE"
docker-compose -f "$COMPOSE_FILE" pull "$SERVICE"

NEW_IMAGE=$(docker-compose -f "$COMPOSE_FILE" config | \
  awk "/^  ${SERVICE}:/{found=1} found && /image:/{print \$2; exit}")

if [[ "$CURRENT_IMAGE" == "$NEW_IMAGE" ]]; then
  echo "No new image available — skipping"
  exit 0
fi

echo "Upgrading $SERVICE: $CURRENT_IMAGE -> $NEW_IMAGE"
docker-compose -f "$COMPOSE_FILE" up -d --no-deps "$SERVICE"

# Poll health endpoint
elapsed=0
while [[ $elapsed -lt $MAX_WAIT ]]; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "$HEALTH_URL" 2>/dev/null || echo "000")
  if [[ "$status" == "200" ]]; then
    echo "Health check passed after ${elapsed}s"
    docker image prune -f
    exit 0
  fi
  sleep 5
  elapsed=$((elapsed + 5))
  echo "Waiting for health check... ${elapsed}s (HTTP $status)"
done

# Rollback
echo "Health check timed out — rolling back to $CURRENT_IMAGE"

# Override image in a temp compose file and re-deploy
sed "s|$NEW_IMAGE|$CURRENT_IMAGE|" "$COMPOSE_FILE" > /tmp/rollback-compose.yml
docker-compose -f /tmp/rollback-compose.yml up -d --no-deps "$SERVICE"

curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \":warning: Deploy FAILED on \`$(hostname)\` for \`$SERVICE\` — reverted to \`$CURRENT_IMAGE\`\"}"

rm /tmp/rollback-compose.yml
exit 1
```

---

## Lambda: Version Aliases with Automatic Traffic Shift

AWS Lambda supports weighted aliases. Deploy to a new version, shift traffic gradually, roll back if error rate rises:

```bash
#!/bin/bash
# lambda-deploy.sh
set -euo pipefail

FUNCTION_NAME="${1:?}"
NEW_ZIP="${2:?}"
REGION="${AWS_DEFAULT_REGION:-us-east-1}"

# Deploy new version
aws lambda update-function-code \
  --function-name "$FUNCTION_NAME" \
  --zip-file "fileb://$NEW_ZIP" \
  --region "$REGION"

aws lambda wait function-updated \
  --function-name "$FUNCTION_NAME" \
  --region "$REGION"

NEW_VERSION=$(aws lambda publish-version \
  --function-name "$FUNCTION_NAME" \
  --region "$REGION" \
  --query 'Version' --output text)

CURRENT_VERSION=$(aws lambda get-alias \
  --function-name "$FUNCTION_NAME" \
  --name production \
  --region "$REGION" \
  --query 'FunctionVersion' --output text 2>/dev/null || echo "1")

echo "New version: $NEW_VERSION, Current: $CURRENT_VERSION"

# Update alias to point 10% traffic at new version
aws lambda update-alias \
  --function-name "$FUNCTION_NAME" \
  --name production \
  --function-version "$CURRENT_VERSION" \
  --routing-config "AdditionalVersionWeights={$NEW_VERSION=0.1}" \
  --region "$REGION"

echo "Observing for 5 minutes at 10% traffic..."
sleep 300

# Check error rate in CloudWatch
ERROR_RATE=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions "Name=FunctionName,Value=$FUNCTION_NAME" "Name=Resource,Value=$FUNCTION_NAME:$NEW_VERSION" \
  --start-time "$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 \
  --statistics Sum \
  --query 'Datapoints[0].Sum // `0`' \
  --output text \
  --region "$REGION")

INVOCATIONS=$(aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions "Name=FunctionName,Value=$FUNCTION_NAME" "Name=Resource,Value=$FUNCTION_NAME:$NEW_VERSION" \
  --start-time "$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 \
  --statistics Sum \
  --query 'Datapoints[0].Sum // `1`' \
  --output text \
  --region "$REGION")

RATE=$(echo "scale=4; $ERROR_RATE / $INVOCATIONS" | bc)
echo "Error rate on new version: $RATE"

if (( $(echo "$RATE > 0.02" | bc -l) )); then
  echo "ERROR RATE TOO HIGH — rolling back"
  aws lambda update-alias \
    --function-name "$FUNCTION_NAME" \
    --name production \
    --function-version "$CURRENT_VERSION" \
    --routing-config 'AdditionalVersionWeights={}' \
    --region "$REGION"
  exit 1
fi

# Promote to 100%
aws lambda update-alias \
  --function-name "$FUNCTION_NAME" \
  --name production \
  --function-version "$NEW_VERSION" \
  --routing-config 'AdditionalVersionWeights={}' \
  --region "$REGION"

echo "Deployment complete: version $NEW_VERSION at 100%"
```

---

## Related Reading

- [How to Set Up Keel for Continuous Delivery](/remote-work-tools/keel-continuous-delivery-setup/)
- [How to Automate Docker Container Updates](/remote-work-tools/automate-docker-container-updates/)
- [How to Set Up ArgoCD for GitOps Workflows](/remote-work-tools/argocd-gitops-workflow-setup/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
