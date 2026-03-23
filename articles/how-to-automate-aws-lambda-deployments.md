---
layout: default
title: "How to Automate AWS Lambda Deployments"
description: "Automate AWS Lambda deployments with SAM, GitHub Actions, and versioned aliases for zero-downtime remote team release workflows"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-aws-lambda-deployments/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Manual Lambda deploys via the AWS console don't scale past one developer. This guide builds a full CI/CD pipeline for Lambda using AWS SAM, GitHub Actions, and CodeDeploy traffic shifting — the same pattern used in production at teams running 50+ functions.

## Prerequisites

- AWS CLI v2 configured (`aws configure`)
- SAM CLI installed
- An S3 bucket for deployment artifacts
- IAM role with Lambda, S3, and CloudFormation permissions

## Project Structure

```
my-lambda-service/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── src/
│   ├── handler.js
│   └── utils.js
├── tests/
│   └── handler.test.js
├── template.yaml          # SAM template
├── samconfig.toml         # environment configs
└── package.json
```

## SAM Template

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Lambda deployment pipeline example

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Runtime: nodejs20.x
    Environment:
      Variables:
        NODE_ENV: !Ref Environment
        LOG_LEVEL: !Ref LogLevel

Parameters:
  Environment:
    Type: String
    AllowedValues: [staging, production]
    Default: staging
  LogLevel:
    Type: String
    Default: info

Resources:
  ApiHandler:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: handler.main
      Description: Main API handler
      AutoPublishAlias: live          # Creates new version on every deploy
      DeploymentPreference:
        Type: Linear10PercentEvery1Minute   # Traffic shifting
        Alarms:
          - !Ref ApiHandlerErrorAlarm
        Hooks:
          PreTraffic: !Ref PreTrafficHook
      Events:
        Api:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY

  PreTrafficHook:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: hooks.preTraffic
      DeploymentPreference:
        Enabled: false
      Policies:
        - Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - codedeploy:PutLifecycleEventHookExecutionStatus
              Resource: '*'

  ApiHandlerErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Lambda error rate alarm
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 2
      Threshold: 5
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref ApiHandler

Outputs:
  FunctionArn:
    Value: !GetAtt ApiHandler.Arn
  ApiEndpoint:
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod"
```

## samconfig.toml

```toml
version = 0.1

[default.global.parameters]
stack_name = "my-lambda-service"

[staging.deploy.parameters]
region = "us-east-1"
s3_bucket = "my-deploy-artifacts-staging"
s3_prefix = "my-lambda-service"
capabilities = "CAPABILITY_IAM"
parameter_overrides = "Environment=staging LogLevel=debug"
confirm_changeset = false
fail_on_empty_changeset = false

[production.deploy.parameters]
region = "us-east-1"
s3_bucket = "my-deploy-artifacts-prod"
s3_prefix = "my-lambda-service"
capabilities = "CAPABILITY_IAM"
parameter_overrides = "Environment=production LogLevel=info"
confirm_changeset = false
fail_on_empty_changeset = false
```

## GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Lambda

on:
  push:
    branches: [main, staging]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

permissions:
  id-token: write    # Required for OIDC
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run tests
        run: npm test -- --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Set environment
        id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "env=production" >> $GITHUB_OUTPUT
          else
            echo "env=staging" >> $GITHUB_OUTPUT
          fi

      - name: SAM build
        run: |
          sam build \
            --config-env ${{ steps.env.outputs.env }} \
            --parallel \
            --cached

      - name: SAM deploy
        run: |
          sam deploy \
            --config-env ${{ steps.env.outputs.env }} \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset

      - name: Get stack outputs
        id: stack
        run: |
          ENDPOINT=$(aws cloudformation describe-stacks \
            --stack-name my-lambda-service \
            --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" \
            --output text)
          echo "endpoint=$ENDPOINT" >> $GITHUB_OUTPUT

      - name: Smoke test
        run: |
          curl -sf "${{ steps.stack.outputs.endpoint }}/health" \
            -H "x-api-key: ${{ secrets.API_KEY }}" \
            || (echo "Smoke test failed" && exit 1)

      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Lambda deploy ${{ job.status }}: ${{ steps.env.outputs.env }} — ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Pre-Traffic Hook

```javascript
// src/hooks.js
const { CodeDeploy } = require('@aws-sdk/client-codedeploy');
const { Lambda } = require('@aws-sdk/client-lambda');

const codedeploy = new CodeDeploy({});
const lambda = new Lambda({});

exports.preTraffic = async (event) => {
  const { DeploymentId, LifecycleEventHookExecutionId } = event;
  let status = 'Succeeded';

  try {
    // Invoke the new version directly (not the alias)
    const functionToTest = process.env.NewVersion;
    const response = await lambda.invoke({
      FunctionName: functionToTest,
      Payload: JSON.stringify({ path: '/health', httpMethod: 'GET' }),
    });

    const body = JSON.parse(Buffer.from(response.Payload).toString());
    if (response.StatusCode !== 200 || body.statusCode !== 200) {
      throw new Error(`Health check failed: ${JSON.stringify(body)}`);
    }

    console.log('Pre-traffic hook passed:', body);
  } catch (err) {
    console.error('Pre-traffic hook failed:', err);
    status = 'Failed';
  }

  await codedeploy.putLifecycleEventHookExecutionStatus({
    deploymentId: DeploymentId,
    lifecycleEventHookExecutionId: LifecycleEventHookExecutionId,
    status,
  });
};
```

## Rollback on Alarm

CodeDeploy automatically rolls back if `ApiHandlerErrorAlarm` triggers during the shift window. To trigger manually:

```bash
# Get the deployment ID
DEPLOY_ID=$(aws deploy list-deployments \
  --application-name my-lambda-service-ApiHandler \
  --deployment-group-name my-lambda-service-ApiHandler-DeploymentGroup \
  --query "deployments[0]" \
  --output text)

# Stop and roll back
aws deploy stop-deployment \
  --deployment-id $DEPLOY_ID \
  --auto-rollback-enabled
```

## Version and Alias Management

```bash
# List all versions
aws lambda list-versions-by-function \
  --function-name my-lambda-service-ApiHandler

# Point alias to a specific version for hotfix
aws lambda update-alias \
  --function-name my-lambda-service-ApiHandler \
  --name live \
  --function-version 42

# Weighted alias for canary (10% to new version)
aws lambda update-alias \
  --function-name my-lambda-service-ApiHandler \
  --name live \
  --routing-config AdditionalVersionWeights={"43"=0.1}

# Promote to 100%
aws lambda update-alias \
  --function-name my-lambda-service-ApiHandler \
  --name live \
  --routing-config AdditionalVersionWeights={}
```

## Local Testing

```bash
# Start local API Gateway emulator
sam local start-api --env-vars env.json --port 3000

# Invoke a single function
sam local invoke ApiHandler \
  --event events/api-request.json \
  --env-vars env.json

# env.json
{
  "ApiHandler": {
    "NODE_ENV": "local",
    "DB_URL": "postgresql://localhost:5432/mydb"
  }
}
```

## Cost and Cold Start Tips

- Set `ProvisionedConcurrencyConfig` in the alias to pre-warm instances for latency-sensitive endpoints
- Use Lambda Powertools for structured logging and X-Ray tracing with minimal overhead
- Enable function URLs with IAM auth instead of API Gateway for internal tooling — no extra cost

```bash
# Add provisioned concurrency after deploy
aws lambda put-provisioned-concurrency-config \
  --function-name my-lambda-service-ApiHandler \
  --qualifier live \
  --provisioned-concurrent-executions 5
```

## Related Reading

- [How to Set Up Semaphore CI for Remote Teams](/how-to-set-up-semaphore-ci-for-remote-teams/)
- [How to Automate Container Image Scanning](/how-to-automate-container-image-scanning/)
- [How to Automate npm Package Publishing](/how-to-automate-npm-package-publishing/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
