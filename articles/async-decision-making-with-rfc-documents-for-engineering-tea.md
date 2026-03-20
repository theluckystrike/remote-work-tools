---
layout: default
title: "Async Decision Making with RFC Documents for Engineering Teams"
description: "A practical guide to implementing async decision making using RFC documents. Includes templates, workflows, and best practices for distributed."
date: 2026-03-16
author: theluckystrike
permalink: /async-decision-making-with-rfc-documents-for-engineering-teams/
categories: [guides]
tags: [async, rfc, decision-making, engineering, remote-work, collaboration]
reviewed: true
intent-checked: true
voice-checked: true
score: 8
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

## Summary
One paragraph explaining the proposal.

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

## Summary
Upgrade our production database from PostgreSQL 13 to 16 to leverage improved query optimization and security patches.

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

## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)
- [How to Create Remote Team Decision Making Framework for.](/remote-work-tools/how-to-create-remote-team-decision-making-framework-for-dist/)
- [Best Wiki Template for Remote Team Engineering Design Documents](/remote-work-tools/best-wiki-template-for-remote-team-engineering-design-docume/)
- [Async Engineering Proposal Process Using GitHub.](/remote-work-tools/async-engineering-proposal-process-using-github-discussions-/)

Built by

Built by theluckystrike — More at [zovo.one](https://zovo.one)
