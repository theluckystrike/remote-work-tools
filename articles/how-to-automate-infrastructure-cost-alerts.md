---
layout: default
title: "How to Automate Infrastructure Cost Alerts"
description: "Set up AWS, GCP, and Azure cost alerts with budget thresholds, anomaly detection, and Slack notifications to prevent cloud bill surprises"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-automate-infrastructure-cost-alerts/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

Cloud cost surprises hit remote teams especially hard — there's no ops person walking the floor noticing an unusual number of running instances. A forgotten load balancer, a misconfigured auto-scaling group, or a runaway batch job can add thousands to your bill before anyone notices. Automated cost alerts turn cost management from a monthly surprise into a real-time signal.

---

## AWS Budget Alerts with Terraform

Define budgets as code so they're version-controlled and applied consistently:

```hcl
# infra/terraform/budgets.tf

# Monthly total cost budget
resource "aws_budgets_budget" "monthly_total" {
  name         = "monthly-total-cost"
  budget_type  = "COST"
  limit_amount = "3000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["ops@yourcompany.com"]
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["ops@yourcompany.com", "cto@yourcompany.com"]
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 110
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["ops@yourcompany.com"]
  }
}

# Per-service budget (catch runaway services early)
resource "aws_budgets_budget" "rds_monthly" {
  name         = "rds-monthly-cost"
  budget_type  = "COST"
  limit_amount = "500"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Service"
    values = ["Amazon Relational Database Service"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 90
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_sns_topic_arns  = [aws_sns_topic.cost_alerts.arn]
  }
}

# SNS topic for routing alerts
resource "aws_sns_topic" "cost_alerts" {
  name = "cost-alerts"
}

# Lambda to forward SNS to Slack
resource "aws_sns_topic_subscription" "cost_alerts_lambda" {
  topic_arn = aws_sns_topic.cost_alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.cost_alert_slack.arn
}
```

Lambda function to forward cost alerts to Slack:

```python
# lambda/cost_alert_slack.py
import json
import urllib.request
import os

SLACK_WEBHOOK = os.environ['SLACK_WEBHOOK_URL']

def lambda_handler(event, context):
    message = event['Records'][0]['Sns']['Message']
    alert_data = json.loads(message)

    budget_name = alert_data.get('budgetName', 'Unknown')
    actual_spend = alert_data.get('actualSpend', {}).get('amount', '?')
    budget_limit = alert_data.get('budgetLimit', {}).get('amount', '?')

    slack_message = {
        "text": f"*AWS Cost Alert*: Budget `{budget_name}`",
        "attachments": [{
            "color": "danger",
            "fields": [
                {"title": "Current Spend", "value": f"${actual_spend}", "short": True},
                {"title": "Budget Limit", "value": f"${budget_limit}", "short": True},
                {"title": "Action", "value": "Review AWS Cost Explorer", "short": False}
            ]
        }]
    }

    req = urllib.request.Request(
        SLACK_WEBHOOK,
        data=json.dumps(slack_message).encode(),
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)
    return {'statusCode': 200}
```

---

## AWS Cost Anomaly Detection

Budget alerts only fire when you exceed a threshold. Cost Anomaly Detection fires when spend is unusual relative to historical patterns, even if you're under budget:

```hcl
# infra/terraform/cost-anomaly.tf

resource "aws_ce_anomaly_monitor" "services" {
  name         = "service-anomaly-monitor"
  monitor_type = "DIMENSIONAL"

  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "ops_team" {
  name      = "ops-team-anomaly-alerts"
  frequency = "DAILY"

  monitor_arn_list = [
    aws_ce_anomaly_monitor.services.arn,
  ]

  subscriber {
    address = aws_sns_topic.cost_alerts.arn
    type    = "SNS"
  }

  threshold_expression {
    or {
      dimension {
        key           = "ANOMALY_TOTAL_IMPACT_PERCENTAGE"
        values        = ["25"]
        match_options = ["GREATER_THAN_OR_EQUAL"]
      }
      dimension {
        key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
        values        = ["50"]
        match_options = ["GREATER_THAN_OR_EQUAL"]
      }
    }
  }
}
```

This alerts when any service has a 25%+ cost increase OR costs jump by $50+ unexpectedly.

---

## Daily Cost Report Script (Multi-Cloud)

Run a daily script that fetches costs and posts a summary to Slack:

```bash
#!/bin/bash
# scripts/daily-cost-report.sh
# Posts yesterday's AWS cost breakdown to Slack

set -euo pipefail

SLACK_WEBHOOK="${SLACK_WEBHOOK}"
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d 2>/dev/null || date -v-1d +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)

# Get total cost
TOTAL=$(aws ce get-cost-and-usage \
  --time-period Start="$YESTERDAY",End="$TODAY" \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --query "ResultsByTime[0].Total.UnblendedCost.Amount" \
  --output text)

# Get top 5 services by cost
TOP_SERVICES=$(aws ce get-cost-and-usage \
  --time-period Start="$YESTERDAY",End="$TODAY" \
  --granularity DAILY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "sort_by(ResultsByTime[0].Groups, &Metrics.UnblendedCost.Amount)[-5:] | reverse(@) | [].[Keys[0], Metrics.UnblendedCost.Amount]" \
  --output text)

# Format message
FIELDS=""
while IFS=$'\t' read -r service cost; do
  cost_rounded=$(printf "%.2f" "$cost")
  FIELDS="${FIELDS},{\"title\":\"$service\",\"value\":\"\$${cost_rounded}\",\"short\":true}"
done <<< "$TOP_SERVICES"

TOTAL_ROUNDED=$(printf "%.2f" "$TOTAL")

curl -s -X POST \
  -H 'Content-type: application/json' \
  --data "{
    \"text\": \"*AWS Daily Cost Report — ${YESTERDAY}*\",
    \"attachments\": [{
      \"color\": \"good\",
      \"fields\": [
        {\"title\": \"Total\", \"value\": \"\$${TOTAL_ROUNDED}\", \"short\": false}
        ${FIELDS}
      ]
    }]
  }" \
  "$SLACK_WEBHOOK"
```

Add to cron:

```bash
# Run at 8am UTC every day
0 8 * * * /opt/scripts/daily-cost-report.sh
```

---

## GCP Budget Alerts

```bash
# Create a budget with Slack notification via gcloud
gcloud billing budgets create \
  --billing-account="01ABCD-23EFGH-456IJK" \
  --display-name="Monthly Budget" \
  --budget-amount="3000USD" \
  --threshold-rule=percent=0.80 \
  --threshold-rule=percent=1.00 \
  --all-updates-rule-pubsub-topic="projects/your-project/topics/billing-alerts"
```

Forward Pub/Sub to Slack via Cloud Function:

```python
# main.py
import base64
import json
import urllib.request
import os

def notify_slack(event, context):
    data = base64.b64decode(event['data']).decode()
    budget_data = json.loads(data)

    cost_amount = budget_data.get('costAmount', 0)
    budget_amount = budget_data.get('budgetAmount', 0)
    threshold = (cost_amount / budget_amount * 100) if budget_amount > 0 else 0

    message = {
        "text": f"*GCP Budget Alert*: {threshold:.0f}% of monthly budget used",
        "attachments": [{
            "color": "danger" if threshold >= 100 else "warning",
            "fields": [
                {"title": "Spent", "value": f"${cost_amount:.2f}", "short": True},
                {"title": "Budget", "value": f"${budget_amount:.2f}", "short": True},
            ]
        }]
    }

    req = urllib.request.Request(
        os.environ['SLACK_WEBHOOK'],
        data=json.dumps(message).encode(),
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)
```

---

## Infracost in CI (Prevent Expensive Changes)

Infracost shows cost impact of Terraform changes in every PR before they're applied:

```yaml
# .github/workflows/infracost.yml
name: Infracost
on: [pull_request]

jobs:
  infracost:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate Infracost diff
        run: |
          infracost diff \
            --path=infra/terraform \
            --format=json \
            --out-file=/tmp/infracost.json \
            --compare-to=HEAD~1
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Post PR comment
        run: |
          infracost comment github \
            --path=/tmp/infracost.json \
            --repo=$GITHUB_REPOSITORY \
            --github-token=${{ secrets.GITHUB_TOKEN }} \
            --pull-request=${{ github.event.pull_request.number }} \
            --behavior=update
```

This posts a comment on each PR showing the monthly cost delta (e.g., "+$47.20/month") before the change is applied.

---

## Related Reading

- [How to Set Up Netdata for Server Monitoring](/remote-work-tools/how-to-set-up-netdata-for-server-monitoring/)
- [How to Create Automated Status Pages](/remote-work-tools/how-to-create-automated-status-pages/)
- [How to Automate Database Backup Verification](/remote-work-tools/how-to-automate-database-backup-verification/)

---

## Related Articles

- [AWS Cost Management for Remote Teams](/remote-work-tools/aws-cost-management-remote-teams-guide/)
- [Remote Engineering Team Infrastructure Cost Per Deploy](/remote-work-tools/remote-engineering-team-infrastructure-cost-per-deploy-track/)
- [Best Tools for Remote Team Daily Health Checks](/remote-work-tools/best-tools-remote-team-daily-health-checks/)
- [Coworking Space Membership vs Day Pass Comparison](/remote-work-tools/coworking-space-membership-vs-day-pass-comparison/)
- [Dubai Remote Work Virtual Visa Cost and Benefits for Tech](/remote-work-tools/dubai-remote-work-virtual-visa-cost-and-benefits-for-tech-pr/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
