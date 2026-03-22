---
layout: default
title: "How to Create Remote Team Style Guides"
description: "Build engineering style guides for remote teams — code style, API design conventions, commit messages, and PR templates with enforcement tooling"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-create-remote-team-style-guides/
categories: [guides]
tags: [remote-work-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Style guides solve a specific remote work problem: code review comments about formatting and naming conventions. In a co-located team, a junior engineer can sit next to a senior and absorb conventions through proximity. In a remote team, they discover them through review feedback at PR time — which is slow and demoralizing. A style guide with automated enforcement eliminates 80% of stylistic review comments, freeing code review time for actual logic.

## What Belongs in a Style Guide

An engineering style guide for remote teams should cover:

1. **Code style** — automated via linters (not manual)
2. **Naming conventions** — not automated, documented
3. **API design patterns** — endpoint naming, error formats, versioning
4. **Commit message format** — automated with commitlint
5. **PR description requirements** — template-enforced
6. **ADR trigger conditions** — when to write an ADR

Anything in the style guide that isn't automated will be inconsistently followed. Prioritize enforcing what you can.

## Code Style: Automate Everything

```yaml
# .github/workflows/lint.yml
name: Code Style
on: [pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install ruff mypy
      - name: Ruff lint
        run: ruff check .
      - name: Ruff format check
        run: ruff format --check .
      - name: Type checking
        run: mypy . --ignore-missing-imports
```

**Python configuration:**

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py312"
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "I",    # isort
    "N",    # naming conventions
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # simplify
]
ignore = ["E501"]  # line too long — handled by formatter

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "auto"

[tool.mypy]
python_version = "3.12"
strict = true
```

**TypeScript/JavaScript:**

```json
// .eslintrc.json
{
  "extends": ["eslint:recommended", "@typescript-eslint/recommended"],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "no-console": "warn",
    "prefer-const": "error"
  }
}
```

```json
// .prettierrc
{
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "semi": true
}
```

## Naming Conventions Document

Conventions that can't be linted must be documented clearly:

```markdown
# Naming Conventions

## Python

### Functions
- Use snake_case for all functions
- Async functions: no prefix — all functions in this codebase may be async
- Boolean functions: prefix with `is_`, `has_`, `can_`, `should_`
  ✅ is_active_user(), has_permission(), can_publish()
  ❌ active_user(), check_permission()

### Classes
- Use PascalCase
- Services: suffix with `Service` (PaymentService, not PaymentManager)
- Repositories: suffix with `Repository` (UserRepository)
- Avoid generic names: Manager, Handler, Util, Helper — be specific

### Variables
- Prefer descriptive names over short names
  ✅ user_count, payment_amount, is_cancelled
  ❌ cnt, amt, flag
- Exception: loop variables i, j, k are fine; keep loop body short

### Constants
- UPPER_SNAKE_CASE
- Group related constants in Enum or TypedDict, not scattered globals

## API Endpoints

- Resource names: plural nouns (/users, /orders, not /user, /order)
- Nested resources: /users/{id}/orders (2 levels max)
- Actions that don't map to CRUD: POST /orders/{id}/cancel
- No verbs in resource names: /orders not /getOrders

## Database

- Table names: plural snake_case (users, order_items)
- Column names: snake_case
- Boolean columns: is_ prefix (is_active, is_deleted)
- Timestamp columns: _at suffix (created_at, deleted_at)
- Foreign keys: {table_singular}_id (user_id, order_id)
- Index names: ix_{table}_{column} or uq_{table}_{column}
```

## Commit Message Convention

Conventional Commits + commitlint:

```bash
# Install commitlint
npm install --save-dev @commitlint/cli @commitlint/config-conventional

# commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat',     // new feature
      'fix',      // bug fix
      'refactor', // code change that neither fixes a bug nor adds a feature
      'perf',     // performance improvement
      'test',     // adding or updating tests
      'docs',     // documentation only
      'ci',       // CI/CD changes
      'chore',    // build process or tooling changes
      'revert',   // revert a previous commit
    ]],
    'subject-max-length': [2, 'always', 72],
    'body-max-line-length': [1, 'always', 100],
  }
};
```

```yaml
# .github/workflows/commitlint.yml
name: Commit Lint
on: [pull_request]
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - run: npm install --save-dev @commitlint/cli @commitlint/config-conventional
      - run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }}
```

**Good commit examples:**

```
feat(auth): add OAuth2 PKCE flow for mobile clients

fix(payment): handle currency conversion rounding in Stripe webhook

refactor(orders): extract order validation into dedicated service

perf(search): add composite index for status+created_at filter

docs(api): document rate limiting headers in OpenAPI spec
```

## PR Description Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->
## Summary
<!-- What does this PR do? 2-3 sentences. Link to the issue or Jira ticket. -->

Closes: #

## Type of change
- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that causes existing functionality to break)
- [ ] Refactoring (no functional changes)
- [ ] Infrastructure / CI change

## Testing
<!-- How was this tested? -->
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Tested locally against staging data
- [ ] Manual QA steps (describe below if applicable)

## Database changes
- [ ] No database changes
- [ ] Migration included — migration is backward compatible
- [ ] Migration included — requires deployment coordination (explain below)

## Checklist
- [ ] Code follows the team style guide
- [ ] Self-review completed
- [ ] Documentation updated (if applicable)
- [ ] No secrets or credentials in code

## Notes for reviewers
<!-- Anything specific you want reviewers to focus on? -->
```

## Publishing the Style Guide

Store the style guide in your documentation repo (or in CONTRIBUTING.md in the main repo):

```markdown
# CONTRIBUTING.md structure

## Quick Start
[How to set up the dev environment in 5 commands]

## Style Guide
[Link to full style guide or inline if short]

## Branching Strategy
- main: always deployable
- feature/: new features, branched from main
- fix/: bug fixes, branched from main
- No long-lived branches

## PR Process
1. Create PR
2. CI checks must pass
3. One approving review required (two for production-critical paths)
4. Squash merge only
5. Delete branch after merge

## ADR Process
[When to write an ADR, where to file it]
```

## Enforcement Without Being Annoying

The key to a style guide that engineers follow is: automate what you enforce strictly, document what you enforce lightly.

**Blocking (CI fails on violation):**
- Code formatting (ruff, prettier)
- Type errors (mypy, TypeScript strict)
- Commit message format (commitlint)
- Security linting (bandit, semgrep rules)

**Non-blocking (warning in CI):**
- Missing docstrings
- Complex function (cyclomatic complexity)
- TODO comments without issue references

**Documented but not enforced:**
- Naming conventions
- PR description quality
- ADR triggers

Trying to enforce naming conventions with AST tools leads to engineer frustration. Document them clearly, mention them in onboarding, and leave them for code review feedback.

## Onboarding New Engineers to the Style Guide

The style guide is useless if new engineers don't know it exists. A structured onboarding checklist is the difference between absorbing conventions in week one versus discovering them through painful PR feedback over three months.

**Onboarding checklist for style guide:**

```markdown
## Engineering Onboarding — Style Guide Checklist

- [ ] Read CONTRIBUTING.md top to bottom
- [ ] Run the linter locally: `make lint` passes on your machine
- [ ] Install pre-commit hooks: `pre-commit install`
- [ ] Read the commit message guide — make your first commit pass commitlint
- [ ] Shadow one code review: observe how comments are structured
- [ ] Submit a PR using the PR template
- [ ] Read the naming conventions doc once (not memorize, just read)
```

**Pre-commit hooks** catch issues locally before they reach CI, which is faster and less demoralizing than a failed CI run:

```bash
# Install pre-commit
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
  - repo: https://github.com/commitizen-tools/commitizen
    rev: v3.16.0
    hooks:
      - id: commitizen
        stages: [commit-msg]
```

Running `pre-commit install` once sets up the hooks. Every commit gets validated before it leaves the developer's machine.

## Style Guide Tooling Comparison

Different team setups warrant different tool choices. Here is a comparison of the most common options across the stack:

| Tool | Language | Enforces | CI Speed | Notes |
|---|---|---|---|---|
| Ruff | Python | Format + lint | Fast | Replaces flake8, isort, black |
| Mypy | Python | Types | Moderate | Strict mode catches most type bugs |
| ESLint | TypeScript/JS | Lint | Fast | Plugin ecosystem is large |
| Prettier | TypeScript/JS | Format | Very fast | Zero config for most projects |
| commitlint | Any | Commit msg | Fast | Requires npm even in Python repos |
| golangci-lint | Go | Lint + format | Fast | Wraps 50+ Go linters |
| RuboCop | Ruby | Lint + format | Moderate | Large rule set, tune carefully |
| Clippy | Rust | Lint | Fast | Built into the Rust toolchain |

For most Python projects, Ruff + Mypy covers 95% of automated enforcement. For TypeScript, ESLint + Prettier + commitlint is the standard setup.

## API Design Conventions in Practice

API consistency problems compound in remote teams. When engineers are not in the same room, they implement endpoints independently and the inconsistencies multiply across services. Document these conventions in the style guide and include worked examples of right versus wrong:

**Error response format — be explicit:**

```json
// Correct: structured error with machine-readable code
{
  "error": {
    "code": "PAYMENT_DECLINED",
    "message": "Payment was declined by the card issuer.",
    "details": {
      "decline_code": "insufficient_funds"
    }
  }
}

// Wrong: freeform string, unstructured
{
  "error": "Something went wrong with your payment"
}
```

**Versioning strategy — pick one and document it:**

- URL versioning (`/v1/users`, `/v2/users`) — most common, easy to route
- Header versioning (`Accept: application/vnd.company.v2+json`) — clean URLs, harder to test in a browser
- Query param (`?version=2`) — avoid; hard to cache and inconsistent

Document which approach your team uses. Engineers creating new endpoints need to know without asking.

## Related Reading

- [Async Code Review Process Without Zoom Calls](/async-code-review-process-without-zoom-calls-step-by-step/)
- [ADR Tools for Remote Engineering Teams](/adr-tools-for-remote-engineering-teams/)
- [Remote Team Deployment Pipeline Best Practices](/remote-team-deployment-pipeline-best-practices/)
---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
