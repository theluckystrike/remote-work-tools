---
layout: default
title: "How to Create Automated Security Scan Pipelines"
description: "Build GitHub Actions pipelines that run SAST, dependency audits, container scans, and secret detection on every PR using Trivy, Semgrep, and Gitleaks configs"
date: 2026-03-22
author: theluckystrike
permalink: /automated-security-scan-pipelines/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, security]
---

{% raw %}
## How to Create Automated Security Scan Pipelines

Security reviews done manually at the end of a sprint find problems after the code is already merged and deployed. Automated scan pipelines catch most issues at PR time, before they land in main, before anyone thinks about deploying them.

This guide builds a layered pipeline: secrets detection, dependency auditing, SAST, and container image scanning.

---

## Layer 1: Secret Detection with Gitleaks

Gitleaks scans commits for API keys, tokens, and credentials before they hit the repository.

**`.github/workflows/gitleaks.yml`**

```yaml
name: Secret Scan
on:
  push:
    branches: ["**"]
  pull_request:

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Custom rules in `.gitleaks.toml`:**

```toml
[extend]
useDefault = true

[[rules]]
id = "internal-api-key"
description = "Internal API key pattern"
regex = '''INTERNAL_[A-Z0-9]{32}'''
tags = ["internal", "api"]

[allowlist]
description = "Allowlist for test files"
paths = [
  '''tests/fixtures/.*''',
]
regexes = [
  '''AKIAIOSFODNN7EXAMPLE''',
]
```

---

## Layer 2: Dependency Auditing

**`dependency-audit.yml`**

```yaml
name: Dependency Audit
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 8 * * 1"

jobs:
  npm-audit:
    runs-on: ubuntu-latest
    if: hashFiles('package-lock.json') != ''
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm audit --audit-level=high

  python-safety:
    runs-on: ubuntu-latest
    if: hashFiles('requirements*.txt') != ''
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install safety
      - run: safety check -r requirements.txt --full-report

  go-vuln:
    runs-on: ubuntu-latest
    if: hashFiles('go.mod') != ''
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "stable"
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - run: govulncheck ./...
```

---

## Layer 3: SAST with Semgrep

**`semgrep.yml`**

```yaml
name: SAST -- Semgrep
on:
  pull_request:
  push:
    branches: [main]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    container:
      image: semgrep/semgrep
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep
        run: |
          semgrep ci \
            --config=p/default \
            --config=p/owasp-top-ten \
            --config=p/secrets \
            --config=.semgrep/custom-rules.yml \
            --sarif \
            --output=semgrep-results.sarif
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Upload SARIF results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep-results.sarif
```

**Custom rules in `.semgrep/custom-rules.yml`:**

```yaml
rules:
  - id: no-hardcoded-credentials
    patterns:
      - pattern: $VAR = "..."
      - metavariable-regex:
          metavariable: $VAR
          regex: (password|secret|api_key|token|passwd)
    message: "Possible hardcoded credential in $VAR"
    languages: [python, javascript, typescript, go]
    severity: ERROR

  - id: sql-string-concat
    pattern: |
      $QUERY = "SELECT " + $VAR
    message: "Possible SQL injection via string concatenation"
    languages: [python, javascript]
    severity: WARNING
```

---

## Layer 4: Container Image Scanning with Trivy

**`container-scan.yml`**

```yaml
name: Container Security Scan
on:
  push:
    branches: [main]
  pull_request:

jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: "CRITICAL,HIGH"
          exit-code: "1"
          ignore-unfixed: true

      - name: Upload Trivy scan results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

      - name: Scan Dockerfile for misconfigurations
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          format: table
          exit-code: "1"
```

---

## Composing the Full Pipeline

```yaml
# .github/workflows/security.yml
name: Security Gate
on:
  pull_request:
  push:
    branches: [main]

jobs:
  secrets:
    uses: ./.github/workflows/gitleaks.yml

  dependencies:
    uses: ./.github/workflows/dependency-audit.yml

  sast:
    uses: ./.github/workflows/semgrep.yml

  container:
    if: hashFiles('Dockerfile') != ''
    uses: ./.github/workflows/container-scan.yml
```

---

## Suppressing False Positives

```bash
# Semgrep inline suppression
password = get_test_fixture_password()  # nosemgrep: no-hardcoded-credentials

# Trivy ignore file
# .trivyignore
CVE-2023-12345

# Gitleaks inline ignore
api_key = "test-key-not-real"  # gitleaks:allow
```

---

## Slack Notification on Failure

```yaml
- name: Notify Slack on security failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": ":rotating_light: Security scan failed on `${{ github.repository }}`",
        "blocks": [{
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": ":rotating_light: *Security scan failed*\nRepo: `${{ github.repository }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View run>"
          }
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_SECURITY_WEBHOOK }}
```

---

## Layer 5: Infrastructure as Code Scanning with Checkov

Terraform, Kubernetes manifests, and Dockerfiles have security misconfigurations that aren't caught by code SAST. Checkov finds them before they reach production.

```yaml
# .github/workflows/checkov.yml
name: IaC Security Scan
on:
  pull_request:
  push:
    branches: [main]

jobs:
  checkov:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov for Terraform
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./terraform
          framework: terraform
          output_format: sarif
          output_file_path: checkov-terraform.sarif
          soft_fail: false
          skip_check: CKV_AWS_79  # Skip specific check if needed

      - name: Run Checkov for Kubernetes manifests
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./k8s
          framework: kubernetes
          output_format: sarif
          output_file_path: checkov-k8s.sarif

      - name: Upload Checkov results
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-terraform.sarif
```

Common Checkov findings to watch:
- S3 buckets without encryption or public access blocks
- Security groups with `0.0.0.0/0` ingress on sensitive ports
- Kubernetes pods running as root without `runAsNonRoot: true`
- RDS instances without deletion protection
- IAM policies with `*` actions

Suppress a finding in Terraform when it's intentional:

```hcl
resource "aws_s3_bucket" "public_assets" {
  # checkov:skip=CKV_AWS_18:Access logging not needed for public static assets
  # checkov:skip=CKV_AWS_21:Versioning not required for public static assets
  bucket = "myapp-public-assets"
}
```

## Enforcing the Security Gate

The pipeline only works as a gate if PR merges are blocked when scans fail. Configure branch protection:

```
Repository → Settings → Branches → Branch protection rules → main
✅ Require status checks to pass before merging
Required checks:
  - Secret Scan / gitleaks
  - Dependency Audit / npm-audit (or python-safety, go-vuln)
  - SAST -- Semgrep / semgrep
  - Container Security Scan / trivy (if Dockerfile exists)
```

For teams using code owners, add a CODEOWNERS rule that requires security team sign-off when any of the scan configuration files change:

```
# .github/CODEOWNERS
.github/workflows/gitleaks.yml    @org/security-team
.github/workflows/semgrep.yml     @org/security-team
.semgrep/                          @org/security-team
.gitleaks.toml                    @org/security-team
terraform/                         @org/security-team
```

Track scan metrics over time by posting results to a dashboard. A rising false-positive rate means rules need tuning. A rising true-positive rate means developers need training on the patterns being caught.

## Related Reading

- [Best Tools for Remote Team Secret Sharing](/remote-work-tools/remote-team-secret-sharing-tools/)
- [How to Automate Pull Request Labeling](/remote-work-tools/automate-pull-request-labeling/)
- [How to Create Automated Dependency Audit](/remote-work-tools/automated-dependency-audit/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
