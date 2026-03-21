---
layout: default
title: "Async Decision Making with RFC Documents for Engineering"
description: "A practical guide to implementing async decision making using RFC documents. Includes templates, workflows, and best practices for distributed"
date: 2026-03-16
author: theluckystrike
permalink: /async-decision-making-with-rfc-documents-for-engineering-teams/
categories: [guides]
tags: [remote-work-tools, async, rfc, decision-making, engineering, remote-work, collaboration]
reviewed: true
intent-checked: true
voice-checked: true
score: 9
---

{% raw %}
# Async Decision Making with RFC Documents for Engineering Teams

Request for Comments (RFC) documents serve as the backbone of asynchronous decision making in distributed engineering organizations. When implemented effectively, RFCs enable teams to make thoughtful, documented decisions without requiring real-time synchronization, which proves especially valuable across time zones.

## What Makes RFCs Effective for Async Decision Making

An RFC document captures a proposed change, its rationale, alternatives considered, and the expected impact. Unlike quick Slack messages or impromptu video calls, RFCs create a persistent, reviewable record that future team members can reference.

The async nature of RFCs provides several key benefits:

- Inclusive participation: Team members contribute feedback on their own schedule
- Thoughtful responses: Reviewers have time to research and craft detailed feedback
- Reduced meeting fatigue: Eliminates the need for synchronous decision-making meetings
- Searchable history: Future teams understand why decisions were made

## Structuring an RFC Document

A well-structured RFC contains specific sections that guide both the author and reviewers through the decision-making process.

```markdown
# RFC: [Short Title]

## Problem Statement
Why is this change needed? What pain point does it address?

## Proposed Solution
Detailed description of the proposed approach.

## Alternatives Considered
What other approaches were evaluated and why they were rejected?

## Implementation Plan
Phases or steps for implementing the decision.

## Open Questions
Issues still requiring discussion or clarification.

## Success Metrics
How will we know this decision was correct?

## Timeline
Key dates and milestones.
```

This structure ensures consistency across proposals and helps reviewers know exactly what to expect.

## Implementing an RFC Workflow

Establishing a clear workflow prevents RFCs from languishing in review limbo. Define explicit time frames for each phase.

### Phase 1: Draft and Initial Feedback

The author creates the RFC and requests initial feedback. Use labels to track status:

```bash
# Example GitHub label workflow
gh label create rfc --color "fbca04" --description "Request for Comments"
gh label create rfc-draft --color "d93f0b" --description "RFC in draft stage"
gh label create rfc-review --color "1d76db" --description "RFC under review"
gh label create rfc-approved --color "0e8a16" --description "RFC approved"
gh label create rfc-closed --color "6e6e6e" --description "RFC not approved"
```

### Phase 2: Review Period

Set a minimum review period—72 hours works well for global teams to accommodate different time zones. During this phase, reviewers add comments, suggest modifications, or express concerns.

Encourage reviewers to use specific markers:

- Question: Clarification needed
- Suggestion: Optional improvement
- Concern: Potential issue requiring resolution
- Approval: Agreement with the proposal

### Phase 3: Decision and Documentation

After the review period, a designated decision-maker (often a tech lead or architect) summarizes feedback and renders a decision. Document the outcome clearly:

```markdown
## Decision

**Status**: Approved / Rejected / Deferred

**Summary of Key Feedback**:
- [Feedback point 1]
- [Feedback point 2]

**Action Items**:
- [ ] Item 1
- [ ] Item 2
```

## Practical Example: Database Migration Decision

Consider a team deciding whether to migrate from PostgreSQL 13 to PostgreSQL 16. An RFC document captures this decision:

```markdown
# RFC: Upgrade PostgreSQL 13 to 16

## Problem Statement
PostgreSQL 13 reaches end-of-life in November 2025. Running unsupported database versions introduces security risks and prevents access to performance improvements.

## Proposed Solution
Perform a blue-green deployment upgrading the primary database to version 16, then promote the new primary.

## Alternatives Considered
1. **Stay on PostgreSQL 13 with extended support**: Costs $5,000/year, delays access to new features
2. **Migrate to managed database service**: Exceeds current budget by 40%

## Implementation Plan
1. Test upgrade in staging environment (Week 1)
2. Schedule maintenance window (Week 2)
3. Perform blue-green deployment (Week 2)
4. Monitor performance metrics (Week 3)

## Success Metrics
- Query performance improvement of 15% or greater
- Zero data loss during migration
- Rollback capability demonstrated in staging
```

This document gives every stakeholder—developers, operations, management—a clear understanding of what changes, why it matters, and what success looks like.

## Managing RFC Review Effectively

Large RFCs overwhelm reviewers. Break complex decisions into smaller, focused documents. If an RFC exceeds 2,000 words, consider splitting it into multiple interconnected RFCs.

Set expectations for response times. A reasonable SLA:

- Initial feedback: 48 hours
- Full review: 72 hours
- Decision announcement: 96 hours from RFC publication

Use asynchronous voting for non-controversial decisions. When an RFC receives no significant concerns after the review period, the default assumption can be approval—explicitly state this in your team conventions.

## Common Pitfalls to Avoid

RFCs fail when they become performative exercises rather than genuine decision-making tools. Avoid these mistakes:

- Proposing without context: Always explain why a decision is needed before proposing a solution
- Ignoring alternatives: Reviewers question proposals without documented alternatives
- Endless revision cycles: Set clear deadlines and stick to them
- No clear owner: Every RFC needs a single author responsible for driving it forward
- Bypassing the process for "urgent" decisions: Reserve exceptions for true emergencies

## RFC Tools and Workflow Integration

Successful RFC processes use tools that make submission and review frictionless:

**Tool Options for Different Team Sizes**

Small teams (5-15):
- GitHub Issues + GitHub Discussions ($0-21/month depending on plan)
- Google Docs with shared folder ($50-140/year for business account)
- Notion with shared workspace ($8-10/user/month)

Mid-size (15-50):
- Dedicated GitHub repo for RFCs ($0 if GitHub-native workflow)
- Slite ($12-15/user/month) with RFC channel
- Confluence (self-hosted free, or $5-8/user/month cloud)

Large teams (50+):
- GitBook ($99-499/month for enterprise)
- Confluence self-hosted (free to download, internal hosting costs)
- Custom internal system (for companies with thousands of RFCs)

**GitHub-based RFC Workflow (Recommended for Technical Teams)**

Use a dedicated `rfcs` repository with this structure:

```
rfcs/
├── text/
│   ├── 0001-new-database-migration.md
│   ├── 0002-microservice-architecture.md
│   └── 0003-auth-system-redesign.md
├── README.md (process overview)
└── decisions/ (merged/accepted RFCs)
    ├── 0001-new-database-migration.md
    └── 0002-microservice-architecture.md
```

Create a pull request for the RFC. GitHub labels and review process handle:
- `rfc-draft` → `rfc-under-review` → `rfc-approved` → `rfc-closed`
- Automated notifications for team members
- Searchable history of all decisions
- Built-in commenting and discussion

## Real RFC Examples

**Example 1: Microservices Migration (Complex, Cross-Team Impact)**

```markdown
# RFC: Migrate from Monolith to Microservices

## Problem Statement
Our monolithic Rails app has reached 200K LOC. Deployments take 45 minutes.
Feature work in one domain blocks unrelated features. Database queries are
slow due to N+1 problems across modules.

## Proposed Solution
Migrate to microservices:
1. Extract payment domain into separate service (3 month timeline)
2. Extract user management domain (2 months)
3. Extract notification domain (1 month)
4. Keep core app for remaining features

## Alternatives Considered
1. Modular monolith with better code organization: Solves structure but
   doesn't improve deployment speed or database query issues
2. Full microservices immediately: Too risky, requires 12-month rewrite
3. Do nothing: Team velocity continues declining 15%/quarter

## Timeline
- Month 1: Design payment service API, set up infrastructure
- Month 2: Extract payment logic, parallel test with existing app
- Month 3: Cutover to payment microservice
- Month 4-6: Extract user management
- Month 7: Extract notifications
- Month 8: Evaluate results, decide next steps

## Risks & Mitigation
- Risk: Distributed systems complexity increases debugging difficulty
  Mitigation: Implement structured logging (ELK stack, $500/month) and
  distributed tracing (Jaeger)

- Risk: Network latency between services impacts performance
  Mitigation: Benchmark in staging, accept <100ms additional latency

## Success Metrics
- Deployment time reduced from 45 to 15 minutes
- New feature time to ship reduced by 30%
- Database query p99 latency reduced by 50%
- Measure at end of each service extraction
```

**Example 2: Engineering Process Change (Moderate Impact, Cross-Team)**

```markdown
# RFC: Implement Code Review SLA

## Problem Statement
Code reviews currently take 24-72 hours. This blocks feature development
and creates context switching when developers return to their code days
later. Team members report frustration waiting for reviews.

## Proposed Solution
Implement 4-hour maximum code review SLA:
- PRs opened during work hours: reviewed within 4 hours
- PRs opened outside work hours: reviewed by 11 AM next business day
- Reviewers add self to PR queue via rotation schedule
- Small PRs (<200 lines) prioritized for faster turnaround

## Implementation Details
1. Create GitHub label: `waiting-review`
2. Implement bot (GitHub Actions, free) that escalates PRs waiting >4h
3. Assign reviewers via rotation (Alice week 1, Bob week 2, etc)
4. Track SLA compliance in weekly metrics

## Risks & Mitigation
- Risk: Forcing fast reviews reduces quality
  Mitigation: "Fast review" doesn't mean "thorough review." Use draft PRs
  and request feedback earlier. Measurement shows quality unchanged.

- Risk: Reviewers get overwhelmed with queue
  Mitigation: Implement PR size guidelines (max 400 lines) first. Large PRs
  get assigned earlier in week.

## Success Metrics
- 90% of PRs reviewed within 4 hours
- Average review time under 8 hours
- Quality metrics (bugs found, regression rate) unchanged
- Team satisfaction survey shows improvement

## Timeline
- Week 1: Communicate change, explain rationale
- Week 2: Deploy bot, establish rotation
- Week 3: Monitor and adjust expectations
- Week 4: Review first weekly metrics
```

## Running Efficient RFC Review Periods

Standard 72-hour review period works well for most teams. However, optimize for:

**Timing considerations:**
- Post RFCs Monday or Tuesday (gives team full week to review)
- Avoid posting Friday afternoon (sits unreviewed over weekend)
- Consider time zones (post 8-9 AM most accessible time zone)

**Participation targets:**
- Minimum 3 reviewers from different teams (ensures diverse perspective)
- Target 1 comment per reviewer (questions, suggestions, concerns)
- Decision-maker provides summary within 24 hours of review period end

**Review efficiency checklist:**
- [ ] Is the problem clearly stated? (Can someone unfamiliar understand?)
- [ ] Are alternatives documented? (Shows author considered options)
- [ ] Is implementation plan realistic? (Can team actually do this?)
- [ ] Are success metrics measurable? (Can you verify it worked?)
- [ ] Are risks identified? (What could go wrong?)

If an RFC is missing any of these, request revisions before starting review period.

## Learning from Decisions

Every approved RFC is a learning opportunity. Monthly, pick one approved RFC and:

1. Review: Was implementation aligned with the RFC?
2. Measure: Did we achieve success metrics?
3. Reflect: What worked? What surprised us? What would we do differently?
4. Document: Add a "Results" section to the RFC with learnings

This practice creates organizational learning that compounds over time. New team members can read old RFCs and understand not just decisions, but the outcomes of those decisions.


## Related Articles

- [Remote Team Async Decision-Making Framework](/remote-work-tools/remote-team-async-decision-making-framework/)
- [Best Practice for Remote Team Decision Making Framework That](/remote-work-tools/best-practice-for-remote-team-decision-making-framework-that/)
- [How to Create Remote Team Decision Making Framework for](/remote-work-tools/how-to-create-remote-team-decision-making-framework-for-dist/)
- [Async Team Retrospective Using Shared Documents and](/remote-work-tools/async-team-retrospective-using-shared-documents-and-recorded/)
- [Remote Team Architecture Decision Record Template for Async](/remote-work-tools/remote-team-architecture-decision-record-template-for-async-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
