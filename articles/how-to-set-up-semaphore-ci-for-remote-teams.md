---
layout: default
title: "How to Set Up Semaphore CI for Remote Teams"
description: "Configure Semaphore CI pipelines with parallel jobs, secrets management, and promotion workflows for distributed engineering teams"
date: 2026-03-22
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-semaphore-ci-for-remote-teams/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, remote-work]
---

{% raw %}

Semaphore CI stands out from GitHub Actions and CircleCI for remote teams because of its native parallelism model, fast machine provisioning, and first-class monorepo support. This guide covers a complete Semaphore setup: pipeline structure, secrets, caching, test parallelism, and environment promotions.

Why Semaphore for Remote Teams

- Monorepo change detection. only run pipelines for changed services
- Self-hosted agents. run jobs inside your own VPC on AWS or GCP
- Pipeline promotions. parameterized deployments with approval gates
- Fast cache. persistent per-branch cache backed by S3

Install the Semaphore CLI

```bash
macOS
brew install semaphoreci/tap/sem

Linux
curl -sL https://storage.googleapis.com/sem-cli-releases/get.sh | bash

Authenticate
sem context create myteam --apikey YOUR_API_KEY

Verify
sem get agents
```

Project Structure

```
my-service/
 .semaphore/
    semaphore.yml        # main pipeline
    deploy-staging.yml   # promotion pipeline
    deploy-prod.yml      # production pipeline
 src/
 tests/
 Dockerfile
```

Main Pipeline

```yaml
.semaphore/semaphore.yml
version: v1.0
name: CI Pipeline

agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2204

auto_cancel:
  running:
    when: "branch != 'main'"

global_job_config:
  prologue:
    commands:
      - checkout
      - cache restore node-modules-$(checksum package-lock.json)
      - npm ci
      - cache store node-modules-$(checksum package-lock.json) node_modules

blocks:
  - name: "Lint & Type Check"
    task:
      jobs:
        - name: ESLint
          commands:
            - npm run lint

        - name: TypeScript
          commands:
            - npm run typecheck

  - name: "Unit Tests"
    task:
      parallelism: 4        # Split tests across 4 workers
      jobs:
        - name: "Test split $SEMAPHORE_JOB_INDEX/$SEMAPHORE_JOB_COUNT"
          commands:
            - npx jest --shard=$SEMAPHORE_JOB_INDEX/$SEMAPHORE_JOB_COUNT \
                       --ci --coverage
            - artifact push workflow coverage-$SEMAPHORE_JOB_INDEX

  - name: "Integration Tests"
    task:
      env_vars:
        - name: DATABASE_URL
          value: "postgresql://postgres:postgres@localhost:5432/test"
      services:
        - name: postgres
          image_name: postgres:16
          env_vars:
            - name: POSTGRES_PASSWORD
              value: postgres
      jobs:
        - name: Integration
          commands:
            - npm run db:migrate
            - npm run test:integration

  - name: "Build"
    run:
      when: "branch = 'main' or tag =~ '.*'"
    task:
      secrets:
        - name: docker-registry
      jobs:
        - name: Docker Build
          commands:
            - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
            - export IMAGE_TAG="${SEMAPHORE_GIT_SHA:0:8}"
            - docker build -t myorg/my-service:$IMAGE_TAG .
            - docker push myorg/my-service:$IMAGE_TAG
            - artifact push workflow image-tag.txt
          after_commands:
            - echo $IMAGE_TAG > image-tag.txt

promotions:
  - name: Deploy to Staging
    pipeline_file: deploy-staging.yml
    auto_promote:
      when: "result = 'passed' and branch = 'main'"

  - name: Deploy to Production
    pipeline_file: deploy-prod.yml
    # No auto_promote. requires manual trigger
```

Staging Deployment Pipeline

```yaml
.semaphore/deploy-staging.yml
version: v1.0
name: Deploy Staging

agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2204

blocks:
  - name: "Deploy"
    task:
      secrets:
        - name: aws-staging
        - name: kubeconfig-staging
      jobs:
        - name: Kubernetes Deploy
          commands:
            - checkout
            - artifact pull workflow image-tag.txt
            - export IMAGE_TAG=$(cat image-tag.txt)
            - kubectl set image deployment/my-service \
                my-service=myorg/my-service:$IMAGE_TAG \
                --namespace=staging
            - kubectl rollout status deployment/my-service \
                --namespace=staging \
                --timeout=300s

  - name: "Smoke Tests"
    task:
      secrets:
        - name: staging-api-key
      jobs:
        - name: Health Check
          commands:
            - |
              for i in {1..10}; do
                curl -sf "https://staging.example.com/health" && break
                sleep 10
              done
        - name: API Smoke Tests
          commands:
            - npm run test:smoke -- --baseUrl=https://staging.example.com
```

Production Pipeline With Approval Gate

```yaml
.semaphore/deploy-prod.yml
version: v1.0
name: Deploy Production

agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2204

blocks:
  - name: "Approval"
    task:
      jobs:
        - name: Awaiting Approval
          commands:
            - echo "Deployment approved by $SEMAPHORE_WORKFLOW_TRIGGERED_BY"

  - name: "Deploy"
    task:
      secrets:
        - name: aws-prod
        - name: kubeconfig-prod
      jobs:
        - name: Kubernetes Deploy
          commands:
            - checkout
            - artifact pull workflow image-tag.txt
            - export IMAGE_TAG=$(cat image-tag.txt)
            - |
              kubectl set image deployment/my-service \
                my-service=myorg/my-service:$IMAGE_TAG \
                --namespace=production
              kubectl rollout status deployment/my-service \
                --namespace=production \
                --timeout=600s

  - name: "Post-Deploy Verification"
    task:
      secrets:
        - name: prod-api-key
      jobs:
        - name: Health Check
          commands:
            - curl -sf "https://api.example.com/health"
        - name: Datadog Event
          commands:
            - |
              curl -s -X POST "https://api.datadoghq.com/api/v1/events" \
                -H "DD-API-KEY: $DATADOG_API_KEY" \
                -H "Content-Type: application/json" \
                -d "{\"title\":\"Production Deploy\",\"text\":\"Deployed $(cat image-tag.txt)\",\"tags\":[\"env:production\"]}"
```

Managing Secrets

```bash
Create a secret from env file
sem create secret aws-staging \
  --env-file .env.staging

Create from individual vars
sem create secret docker-registry \
  -e DOCKER_USERNAME=myteam \
  -e DOCKER_PASSWORD=supersecret

Create from file (e.g., kubeconfig)
sem create secret kubeconfig-prod \
  -f kubeconfig.yaml:/root/.kube/config

List secrets
sem get secrets

Update a secret
sem edit secret aws-staging
```

Monorepo Change Detection

```yaml
.semaphore/semaphore.yml. monorepo variant
version: v1.0
name: Monorepo CI

agent:
  machine:
    type: e1-standard-2
    os_image: ubuntu2204

blocks:
  - name: "Service A"
    run:
      when: "change_in('/services/service-a/', {default_branch: 'main'})"
    task:
      jobs:
        - name: Test
          commands:
            - checkout
            - cd services/service-a
            - npm ci && npm test

  - name: "Service B"
    run:
      when: "change_in('/services/service-b/', {default_branch: 'main'})"
    task:
      jobs:
        - name: Test
          commands:
            - checkout
            - cd services/service-b
            - pip install -r requirements.txt && pytest

  - name: "Shared Infrastructure"
    run:
      when: "change_in('/infrastructure/', {default_branch: 'main'})"
    task:
      jobs:
        - name: Terraform Plan
          commands:
            - checkout
            - cd infrastructure
            - terraform init && terraform plan
```

Self-Hosted Agent on AWS

```bash
On the EC2 instance (Ubuntu 22.04)
curl -sL https://storage.googleapis.com/sem-cli-releases/get.sh | bash

Register agent
sem agent register \
  --endpoint https://myteam.semaphoreci.com \
  --token $AGENT_TOKEN \
  --name aws-agent-01 \
  --type s1-prod-large

Install as systemd service
sem agent install --start-on-boot
sudo systemctl start semaphore-agent
```

Use self-hosted agents in pipelines:

```yaml
agent:
  machine:
    type: s1-prod-large    # your registered agent type
```

Caching Strategy

```yaml
Efficient cache for Node.js
- cache restore node-modules-${{ checksum "package-lock.json" }}
- npm ci
- cache store node-modules-${{ checksum "package-lock.json" }} node_modules

Docker layer caching
- cache restore docker-layers
- docker build --cache-from myorg/my-service:cache \
    --build-arg BUILDKIT_INLINE_CACHE=1 \
    -t myorg/my-service:cache \
    -t myorg/my-service:$IMAGE_TAG .
- docker push myorg/my-service:cache
- cache store docker-layers ~/.docker/buildx
```

Notifications

```yaml
.semaphore/semaphore.yml. add at the top level
notifications:
  - name: Slack Failures
    rules:
      - when: "result = 'failed'"
    notify:
      slack:
        endpoint: https://hooks.slack.com/services/YOUR/WEBHOOK
        channels: ["#ci-alerts"]
        message: "Pipeline failed: {{.WorkflowName}}. {{.Revision}}"
```

Related Reading

- [How to Automate AWS Lambda Deployments](/how-to-automate-aws-lambda-deployments/)
- [How to Set Up Flux CD for GitOps](/how-to-set-up-flux-cd-for-gitops/)
- [ArgoCD GitOps Workflow Setup](/argocd-gitops-workflow-setup/)

---

Built by theluckystrike. More at [zovo.one](https://zovo.one)

{% endraw %}
