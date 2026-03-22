---
layout: default
title: "Async Engineering Proposal Process Using GitHub Discussions"
description: "A practical guide to running async engineering proposals using GitHub Discussions. Includes setup steps, templates, and automation tips for distributed"
date: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /async-engineering-proposal-process-using-github-discussions-/
categories: [guides]
tags: [remote-work-tools, async, github, proposals, engineering, remote-work]
reviewed: true
intent-checked: true
voice-checked: true
score: 8
---

Engineering teams working across time zones cannot rely on synchronous meetings to make technical decisions. A proposal left pending until everyone is online means decisions blocked for days. GitHub Discussions solves this by giving engineering proposals a permanent, searchable home where reviewers engage on their own schedules and decisions are documented where the code lives.

This guide walks through setting up a complete async engineering proposal process using GitHub Discussions — from repository configuration to automation and day-to-day workflow.

## Why GitHub Discussions for Engineering Proposals

Most teams use some combination of Slack threads, Google Docs, or Confluence pages for technical proposals. These approaches share a common problem: the discussion is disconnected from the code. A proposal approved in a Confluence comment thread has no git history, no association with the resulting PR, and no searchability from within the repository.

GitHub Discussions keeps proposals in the same namespace as the work they produce. When a proposal results in a PR, you can reference the Discussion directly. When a new engineer joins and wonders why a specific architectural decision was made, they can find the original proposal by searching the repository.

GitHub Discussions also has built-in features that map well to async proposal workflows: upvotes for sentiment, the "Answer" marker for resolved questions, pinning for active proposals, and label-based categorization for filtering.

## Step 1: Enable and Configure GitHub Discussions

Navigate to your repository Settings, scroll to the Features section, and enable Discussions. Once enabled, go to the Discussions tab and create a dedicated category:

- **Name:** Engineering Proposals
- **Description:** Technical proposals, RFCs, and architecture decisions
- **Format:** Open-ended discussion (not Q&A format)

Create additional categories if useful:
- **ADRs (Architecture Decision Records)** — for final, approved decisions worth preserving
- **Explorations** — for early-stage ideas that aren't formal proposals yet

## Step 2: Create the Proposal Template

GitHub Discussions supports Discussion templates via `.github/DISCUSSION_TEMPLATE/` directory. Create a file at `.github/DISCUSSION_TEMPLATE/engineering-proposal.yml`:

```yaml
title: "[Proposal] "
labels: ["pending-review"]
body:
  - type: markdown
    attributes:
      value: |
        Use this template for all engineering proposals. Fill every section before submitting.
  - type: textarea
    id: problem
    attributes:
      label: Problem Statement
      description: What specific problem does this solve? Include metrics or user impact.
    validations:
      required: true
  - type: textarea
    id: solution
    attributes:
      label: Proposed Solution
      description: Detailed technical description with code examples, diagrams, or architecture changes.
    validations:
      required: true
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives Considered
      description: What other approaches did you evaluate and why were they rejected?
    validations:
      required: true
  - type: textarea
    id: impact
    attributes:
      label: Impact Assessment
      description: "Breaking changes, migration required, performance implications, security considerations."
    validations:
      required: true
  - type: textarea
    id: questions
    attributes:
      label: Open Questions
      description: Decisions you are unsure about and want reviewer input on.
  - type: input
    id: deadline
    attributes:
      label: Review Deadline
      placeholder: "e.g., 2026-03-28"
    validations:
      required: true
  - type: input
    id: reviewers
    attributes:
      label: Required Reviewers
      placeholder: "@alice @bob"
    validations:
      required: true
```

This template guarantees every proposal follows a reviewable structure and prevents the most common failure mode: sparse proposals that get sparse feedback.

## Step 3: Set Up Review Workflow Automation

Create a GitHub Actions workflow to manage proposal lifecycle. Save as `.github/workflows/proposal-review.yml`:

```yaml
name: Engineering Proposal Review

on:
 discussion:
 types: [created, edited]

jobs:
 label-proposal:
 runs-on: ubuntu-latest
 steps:
 - uses: actions/github-script@v6
 with:
 script: |
 const discussion = context.payload.discussion;
 if (discussion.category.name === 'Engineering Proposals') {
 github.rest.issues.addLabels({
 owner: context.repo.owner,
 repo: context.repo.repo,
 issue_number: discussion.number,
 labels: ['pending-review']
 });

 // Create review deadline (7 days from now)
 const deadline = new Date();
 deadline.setDate(deadline.getDate() + 7);

 github.rest.discussions.update({
 owner: context.repo.owner,
 repo: context.repo.repo,
 discussion_number: discussion.number,
 category_id: discussion.category.id,
 pinned: true
 });
 }
```

This workflow automatically labels new proposals and pins them for visibility.

## Step 4: Running the Proposal Process

With infrastructure in place, here's how to run an actual proposal:

### Phase 1: Draft and Submit

1. Create a new Discussion using the Engineering Proposal template
2. Fill in all sections thoroughly — sparse proposals get sparse feedback
3. Add specific reviewers using the **Required Reviewers** field
4. Set a review deadline (typically 5-7 business days)
5. Post a link in your team's Slack channel with a brief one-line summary

### Phase 2: Async Review

Reviewers engage on their own schedules. Encourage them to use:

- Comments for questions, concerns, or suggestions
- Reactions for quick acknowledgment ("I read this")
- The Answer marker to mark specific questions as resolved
- Upvotes and downvotes for sentiment on specific approaches

Reviewers from different time zones will engage at different points in the review window. This is fine — async proposals are designed to accumulate feedback over time, not demand simultaneous attention.

### Phase 3: Collect Decisions

After the review period, the proposal author summarizes feedback as a pinned comment:

```markdown
## Decision Summary

### Consensus Reached
- Team agrees on using PostgreSQL for the new data layer
- Migration will happen in phases over 2 sprints

### Open Items Requiring Follow-up
- Caching strategy needs additional analysis
- Need security review before implementation

### Final Decision
**Approved** with the conditions above. Moving to implementation.
Implementation tracking: [Link to GitHub Issue]
```

Marking this comment as the Answer closes the proposal cleanly and makes the decision findable later.

## Step 5: Automate Status Updates

Track proposal status by creating a simple label-based system:

| Label | Meaning |
|-------|---------|
| `pending-review` | Awaiting team feedback |
| `changes-requested` | Needs revision before approval |
| `approved` | Accepted and ready for implementation |
| `rejected` | Not moving forward |
| `superseded` | Replaced by another proposal |

Use GitHub Issues to create tracking items for approved proposals:

```yaml
# After proposal approval, create tracking issue
github.rest.issues.create({
 owner: context.repo.owner,
 repo: context.repo.repo,
 title: "[Tracking] Implementation: New Data Layer Migration",
 body: "Tracking issue for approved proposal: [Link to discussion]\n\n- [ ] Phase 1: Schema migration\n- [ ] Phase 2: Data migration scripts\n- [ ] Phase 3: Application updates",
 labels: ['implementation', 'tracking']
});
```

## Best Practices for Effective Async Proposals

### Write for Busy Reviewers

Your proposal competes with actual coding work. Make it easy to consume:

- Lead with the summary — busy reviewers read this first
- Use code blocks for technical details
- Include visual diagrams when architecture changes are involved
- Keep proposals focused (split large proposals into multiple RFCs if needed)

### Set Clear Deadlines

Without explicit deadlines, proposals drift indefinitely. Include:

```markdown
**Review period**: March 16-23, 2026
**Decision target**: March 24, 2026
```

Follow up with a Slack reminder two days before the deadline if key reviewers haven't weighed in.

### Respect Time Zones

For global teams:

- Avoid requiring responses within 24 hours
- Allow 2-3 full working days for substantive feedback
- Use UTC times in deadline announcements
- Never schedule a "proposal decision call" that requires night-shift attendance from any timezone

### Close the Loop

Every proposal deserves a final update:

- **Accepted:** Link to implementation tracking issue
- **Rejected:** Explain why and whether the door remains open for future attempts
- **Deferred:** Note conditions that would make this viable later

Proposals that quietly disappear erode trust in the process. Even a "we decided not to do this because X" is valuable documentation.

## Measuring Proposal Process Effectiveness

Track these metrics to improve your async proposal process over time:

- **Time to decision:** Average days from submission to resolution
- **Review participation rate:** What percentage of the team engages with each proposal?
- **Revision cycles:** How many proposals require major revision before approval?
- **Implementation rate:** How many approved proposals actually ship within a quarter?

Use GitHub's built-in analytics or export Discussion data via the API for analysis. A healthy process typically sees decisions within 7 days and participation from at least half the core team on significant proposals.


## Related Articles

- [Async Capacity Planning Process for Remote Engineering](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-manag/)
- [Async Capacity Planning Process for Remote Engineering — Managers](/remote-work-tools/async-capacity-planning-process-for-remote-engineering-managers-guide/)
- [Async Release Notes Writing Process for Distributed](/remote-work-tools/async-release-notes-writing-process-for-distributed-engineering-teams/)
- [.github/ISSUE_TEMPLATE/onboarding.yml](/remote-work-tools/hybrid-team-onboarding-process-template-for-new-hires-splitting-time-office-and-home/)
- [Async Pair Programming Workflow Using Recorded Walkthroughs](/remote-work-tools/async-pair-programming-workflow-using-recorded-walkthroughs-and-github/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
