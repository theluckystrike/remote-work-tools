---

layout: default
title: "Async Engineering Proposal Process Using GitHub Discussions Step by Step"
description: "A practical guide to running async engineering proposals using GitHub Discussions. Includes setup steps, templates, and automation tips for distributed teams."
date: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /async-engineering-proposal-process-using-github-discussions-/
categories: [guides]
tags: [async, github, proposals, engineering, remote-work]
reviewed: true
intent-checked: false
voice-checked: false
score: 8
---


{% raw %}
# Async Engineering Proposal Process Using GitHub Discussions Step by Step

Engineering proposals in distributed teams face a fundamental challenge: getting meaningful input from reviewers across multiple time zones without scheduling synchronous meetings. GitHub Discussions provides a powerful, integrated solution for async proposal reviews that keeps all conversations searchable, transparent, and well-organized.

This guide walks you through setting up and running an async engineering proposal process using GitHub Discussions, from initial setup to final decision tracking.

## Why GitHub Discussions for Engineering Proposals

GitHub Discussions offers several advantages over traditional proposal methods:

- **Integrated with your workflow**: Proposals live in the same repository where code changes happen
- **Asynchronous by default**: Team members contribute on their own schedules
- **Searchable**: Future teams can find past decisions and reasoning
- **Threaded conversations**: Related discussions stay organized
- **Voting mechanisms**: Easy sentiment gathering on proposals

Unlike RFCs buried in Google Docs or Notion, Discussions connect directly to issues and pull requests, creating a complete audit trail of why decisions were made.

## Step 1: Create a Proposal Category

Start by setting up a dedicated space for engineering proposals in your repository.

1. Navigate to your repository on GitHub
2. Click the **Discussions** tab
3. Click **Edit repository settings** (the gear icon)
4. Under "Discussion category settings," add a new category called **Engineering Proposals**
5. Configure it with these settings:
   - **Format**: [x] Announcement (for pinned proposals)
   - **Emoji**: 📋

You can also create supporting categories like **Decision Archive** (for accepted proposals) and **Questions** (for early-stage brainstorming).

## Step 2: Define a Proposal Template

A consistent template ensures every proposal contains the information reviewers need. Create `.github/DISCUSSION_TEMPLATE/engineering-proposal.md`:

```yaml
---
name: Engineering Proposal
about: Submit an engineering proposal for team review
title: "[Proposal] "
labels: proposal
assignees: ''
---

## Summary

<!-- 2-3 sentence description of what you're proposing and why it matters -->

## Problem Statement

<!-- What specific problem does this solve? Include metrics or user impact if available -->

## Proposed Solution

<!-- Detailed technical description with code examples, diagrams, or architecture changes -->

## Alternatives Considered

<!-- What other approaches did you evaluate? Why were they rejected? -->

## Impact Assessment

- **Breaking changes**: 
- **Migration required**: 
- **Performance implications**: 
- **Security considerations**: 

## Open Questions

<!-- Decisions you're unsure about and want reviewer input on -->

## Timeline

<!-- Estimated implementation phases if approved -->

---

**Submitted by**: 
**Review deadline**: 
**Required reviewers**: 
```

This template guarantees every proposal follows a reviewable structure.

## Step 3: Set Up Review Workflow

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
2. Fill in all sections thoroughly—sparse proposals get sparse feedback
3. Add specific reviewers using the **Reviewers** section
4. Set a review deadline (typically 5-7 business days)

### Phase 2: Async Review

Reviewers engage on their own schedules. Encourage them to use:

- **Comments**: For questions, concerns, or suggestions
- **💬 reactions**: Quick acknowledgment ("I read this")
- **Answer (✓)**: Marking questions as resolved
- **Upvote/Downvote**: For sentiment on specific approaches

### Phase 3: Collect Decisions

After the review period, the proposal author summarizes feedback:

```markdown
## Decision Summary

### Consensus Reached
- ✅ Team agrees on using PostgreSQL for the new data layer
- ✅ Migration will happen in phases over 2 sprints

### Open Items Requiring Follow-up
- ❓ Caching strategy needs additional analysis
- ❓ Need security review before implementation

### Final Decision
**Approved** with the conditions above. Moving to implementation.
```

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

- Lead with the summary—busy reviewers read this first
- Use code blocks for technical details
- Include visual diagrams when architecture changes are involved
- Keep proposals focused (split large proposals into multiple RFCs)

### Set Clear Deadlines

Without explicit deadlines, proposals drift. Include:

```markdown
**Review period**: March 16-23, 2026
**Decision target**: March 24, 2026
```

### Respect Time Zones

For global teams:

- Avoid requiring responses within 24 hours
- Allow 2-3 full working days for substantive feedback
- Use UTC times in deadline announcements

### Close the Loop

Every proposal deserves a final update:

- Accepted: Link to implementation tracking issue
- Rejected: Explain why and whether the door remains open for future attempts
- Deferred: Note conditions that would make this viable later

## Measuring Proposal Process Effectiveness

Track these metrics to improve your async proposal process:

- **Time to decision**: Average days from submission to resolution
- **Review participation**: What percentage of team engages?
- **Revision cycles**: How many proposals need major revisions?
- **Implementation rate**: How many approved proposals actually ship?

Use GitHub's built-in analytics or export Discussion data to a spreadsheet for analysis.

## Conclusion

GitHub Discussions provide a robust foundation for async engineering proposals in distributed teams. By establishing clear categories, templates, and workflows, you create a process that scales across time zones while maintaining thorough documentation of technical decisions.

The key is consistency: every proposal follows the same structure, every review gets acknowledged, and every decision gets documented. This builds institutional knowledge over time while keeping your team productive without synchronous meetings.

Start small with one category and one template, then refine based on what works for your team's specific needs.

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
