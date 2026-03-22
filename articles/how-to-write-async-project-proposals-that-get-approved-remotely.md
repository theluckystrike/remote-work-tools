---
layout: default
title: "How to Write Async Project Proposals That Get Approved"
description: "A practical guide to crafting async project proposals that get approved in remote teams. Learn frameworks, templates, and strategies for winning async"
date: 2026-03-18
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-write-async-project-proposals-that-get-approved-remotely/
categories: [guides]
tags: [remote-work-tools, project-management, remote-work, async, proposals, approval-workflow, decision-making]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
---

{% raw %}

Getting buy-in on projects without the benefit of face-to-face conversation or real-time discussion is one of the hardest skills to develop in remote work. When you can't walk into a manager's office, can't read body language, and can't immediately address questions, your proposal document needs to do all the heavy lifting. Async project proposals that get approved remotely share common characteristics: they're clear, anticipate objections, provide all necessary context, and make decision-making easy for reviewers.

This guide walks through the anatomy of effective async project proposals, frameworks that work across different team structures, and practical tips for improving your approval rates without ever scheduling a meeting.

## Why Async Proposals Are Different From In-Person Pitches

When you present a project proposal in a meeting, you have several advantages that disappear in async communication. You can read the room and adjust your pitch in real-time. You can answer questions immediately. You can gauge interest through body language and adjust your approach on the fly. You can dig deeper into objections as they arise.

Async proposals must be self-contained documents that anticipate every reasonable question a reviewer might have. They must be persuasive even when the author isn't present to defend them. This constraint actually produces better proposals—in-person pitches often gloss over weaknesses or rely on charisma to mask gaps. Async proposals force you to address those weaknesses directly, which builds trust and leads to better project outcomes.

The other challenge is attention. Your reviewer is likely reading your proposal between meetings, during a busy day, or at the end of a long week. They won't give you their undivided attention, so your proposal needs to deliver maximum value with minimum time investment.

## Prerequisites

Before you begin, make sure you have the following ready:

- A computer running macOS, Linux, or Windows
- Terminal or command-line access
- Administrator or sudo privileges (for system-level changes)
- A stable internet connection for downloading tools


### Step 1: The Anatomy of a Winning Async Proposal

Every successful async project proposal contains several key sections. Skipping any of these reduces your chances of approval.

### Step 2: Frameworks for Different Types of Proposals

Not all proposals are the same. Adjust your approach based on what you're asking for.

### Small Projects or Experiments

For low-risk initiatives under $5,000 or 2 weeks of engineering time, keep your proposal brief. Focus on the problem, your solution, and why you're confident it will work. Don't over-document—speed matters for experiments, and your reviewer knows this.

### Major Initiatives

Large projects require more rigorous documentation. Include detailed cost-benefit analysis, stakeholder buy-in evidence, and explicit sign-off from dependent teams. Consider creating a companion slide deck or Loom video for complex concepts that are hard to convey in text.

### Process Changes

Proposals that change how the team works require extra attention to buy-in. Document current pain points with specific examples. Identify all affected parties and show you've gathered input from them. Anticipate resistance and address it directly.

### Step 3: Writing Tips That Increase Approval Rates

### Lead with Outcomes, Not Activities

Reviewers care about results, not tasks. Instead of "We will conduct user research and create wireframes," say "We will reduce onboarding time by 40% by simplifying the signup flow based on user research insights."

### Use Visuals Strategically

A well-placed diagram, flowchart, or mockup can convey what would take paragraphs of text. Include visuals for complex workflows, architectural changes, or UI updates. Keep them simple—your reviewer is scanning, not studying.

### Address the Money Directly

If your proposal requires budget, address cost explicitly. Don't make your reviewer hunt for the price. Include ROI calculations where possible. If you can't quantify ROI, explain the strategic value in clear terms.

### Make It Easy to Say Yes

Include a clear approval mechanism. Can they just reply "LGTM"? Should they comment on specific sections? Do you need a formal sign-off in a tool like Asana or Jira? Reduce friction at every step.

### Follow Up Strategically

After sending your proposal, follow up at the right time. 3-5 days is appropriate for most proposals. Your follow-up should be brief—just a reminder with any new context. Avoid being pushy or making the reviewer feel pressured.

### Step 4: Common Mistakes That Kill Proposals

**Being too vague.** Proposals that say "improve the product" without specifics fail. Every claim should have evidence or a clear path to evidence.

**Ignoring trade-offs.** Every project has costs—opportunity costs, technical debt, time away from other work. Acknowledging trade-offs shows you understand the full picture.

**Asking for too much at once.** If your big vision requires multiple approvals, consider breaking it into phases. Get approval for phase one, then come back with proof of concept for phase two.

**Not knowing your audience.** Different stakeholders care about different things. Executives care about business impact. Engineering managers care about feasibility and team capacity. Write for your specific reviewer.

**Sending and disappearing.** Async doesn't mean hands-off. Be available for questions, respond promptly to comments, and show engagement with feedback.

### Step 5: Templates You Can Adapt

Here's a template structure you can customize for your team's needs:

```
### Step 6: Problem
[What issue are you solving? Evidence?]

### Step 7: Solution
[Your proposed approach]

### Step 8: Impact
[Expected outcomes with metrics]

### Step 9: Timeline
[Key milestones and dates]

### Step 10: Resources Needed
[Budget, people, access]

### Step 11: Risks & Mitigation
[Key risks and how you'll address them]

### Step 12: Ask
[What you need and by when]
```

### Step 13: Build Your Reputation for Future Proposals

Your approval rate improves over time as you build credibility. Deliver on your promises. When projects succeed, document the results and share them. When they don't succeed, analyze what went wrong and share those learnings too.

Proposals from someone with a track record of successful projects get more trust and faster approvals than proposals from someone unknown. Think of each proposal as an investment in your future influence.

### Step 14: Jira Automation Scripts for Remote Teams

Automating Jira ticket creation and status updates reduces administrative overhead in distributed teams.

```python
import requests
from requests.auth import HTTPBasicAuth
import json
from datetime import datetime, timedelta

JIRA_URL = "https://your-org.atlassian.net"
AUTH = HTTPBasicAuth("email@example.com", "YOUR_API_TOKEN")
HEADERS = {"Accept": "application/json", "Content-Type": "application/json"}

def create_ticket(project_key, summary, description, issue_type="Task", assignee=None):
    payload = {
        "fields": {
            "project": {"key": project_key},
            "summary": summary,
            "description": {
                "type": "doc", "version": 1,
                "content": [{"type": "paragraph", "content":
                    [{"type": "text", "text": description}]}]
            },
            "issuetype": {"name": issue_type},
        }
    }
    if assignee:
        payload["fields"]["assignee"] = {"accountId": assignee}
    r = requests.post(
        f"{JIRA_URL}/rest/api/3/issue",
        auth=AUTH, headers=HEADERS,
        data=json.dumps(payload)
    )
    return r.json()

def get_overdue_tickets(project_key, days_overdue=3):
    cutoff = (datetime.now() - timedelta(days=days_overdue)).strftime("%Y-%m-%d")
    jql = (f"project = {project_key} AND status != Done "
           f"AND due <= '{cutoff}' ORDER BY due ASC")
    r = requests.get(
        f"{JIRA_URL}/rest/api/3/search",
        auth=AUTH, headers=HEADERS,
        params={"jql": jql, "fields": "summary,assignee,due,status", "maxResults": 50}
    )
    return r.json()["issues"]
```

Generate API tokens at id.atlassian.net/manage-profile/security/api-tokens. Tokens are scoped to the user's permissions — use a service account for shared automation.

## Proposal Management Tools Comparison

Several platforms automate proposal workflow and increase approval rates:

**Google Docs (Free)**
- Best for: Small teams, simple proposals
- Approval flow: Comments + edit suggestions, manual sign-off
- Versioning: Auto-saves, simple revision history
- Collaboration: Real-time co-editing, comment threads
- Drawbacks: No formal approval workflow, relies on email follow-up

Set up template with standard sections (Problem, Solution, Impact, Ask) — reuse across projects.

**PandaDoc (Proposal + Signature)**
- Cost: $30-100/month
- Features: Beautiful templates, e-signature, approval workflows, tracking
- Integrates: Slack, Salesforce, HubSpot
- Approval visibility: Dashboard shows who opened/reviewed/signed
- Great for client-facing proposals that need signatures

Template example:
```
1. Executive Summary (1 paragraph)
2. Current State Analysis (data-backed)
3. Proposed Solution (detailed, visual)
4. Timeline & Phases (Gantt chart)
5. Resource Allocation (team assignments)
6. Budget & ROI (transparent pricing)
7. Success Metrics (measurable KPIs)
8. Risk & Mitigation
9. Approval Sign-off (e-signature)
```

**Notion (Internal Proposals)**
- Cost: Free (Notion Teams plan $10/user/month)
- Database approach: Each proposal is trackable entity
- Status workflow: Draft → Review → Approved → In Progress
- Comments: Thread-based discussion directly on proposal
- Templates: Reusable across all project types

Database properties track:
- Status (Draft, Under Review, Approved, Rejected, In Progress)
- Reviewer assigned (manager, stakeholder)
- Estimated cost
- Expected ROI %
- Decision date
- Comments/feedback

**Craft (Apple Ecosystem)**
- Cost: $12/month for Teams
- Best for: Teams already in Apple ecosystem
- Formatting: Excellent design templates
- Collaboration: Invite reviewers with comment access
- Mobile: Works great on iPad for on-the-go review

**Linear Proposal Tracking (For Technical Projects)**
- Native in Linear issue tracking
- Link proposals to GitHub PRs, architectural decisions
- Discussion threads within Linear
- Approval state tracked with stakeholder sign-off
- Integrates directly with sprint planning

### Step 15: Proposal Templates by Project Type

### Small Feature/Experiment (Under $5K, 2 weeks)
```markdown
# [Feature Name] Proposal

### Step 16: Problem
[1-2 sentences describing user pain or business gap]
Data: [one metric showing impact]

### Step 17: Solution
[3-5 sentences of approach]

### Step 18: Implementation
- Timeline: X days
- Team: [who]
- Dependencies: [list any blockers]

### Step 19: Metrics for Success
- Launch date: [date]
- Success criteria: [measurable outcome]

### Step 20: Ask
[Explicit approval request: budget, timeline, resources]
```

Keep this under 1 page. Decision should take <5 minutes.

### Medium Initiative (2-8 weeks, $5-50K)
```markdown
# [Initiative] Proposal

## Executive Summary
[2-3 sentence overview of what, why, expected impact]

### Step 21: Current Problem Analysis
- Quantified pain point (metrics)
- Root cause analysis
- Cost of inaction (financial impact)

### Step 22: Proposed Solution
- High-level approach (diagram if helpful)
- Phase breakdown with deliverables
- Technical approach (brief)
- Team composition and skills needed

### Step 23: Business Impact
- Revenue impact (if applicable)
- Cost savings
- Risk reduction
- User satisfaction improvement

### Step 24: Timeline
| Phase | Deliverable | Duration | Owner |
|-------|-------------|----------|-------|
| 1 | [spec] | 2w | [name] |
| 2 | [build] | 3w | [name] |
| 3 | [launch] | 1w | [name] |

## Resource Requirements
- Engineering: X FTE for Y weeks
- Product: Z days for planning
- Design: A days
- Budget: $$$

### Step 25: Risks & Mitigation
- Risk 1: [scenario], Mitigation: [action]
- Risk 2: [scenario], Mitigation: [action]

### Step 26: Success Metrics
- Launch goal: [date]
- Adoption target: [%]
- Quality gates: [criteria]

### Step 27: Decision Needed
Approve budget + timeline by [date]
```

### Major Strategic Initiative (>8 weeks, >$50K)
Add to the medium template:
- Competitive analysis (why now?)
- Alternative approaches considered + why rejected
- Executive stakeholder buy-in evidence
- Financial modeling (3-year projection)
- Risk analysis with contingency budget
- Quarterly milestone reviews built in

### Step 28: Approval Optimization Techniques

**Pre-Submit Review Checklist**
Before sending to decision-makers, validate:

```bash
# Proposal quality checklist
- [ ] Problem quantified with data (not opinions)
- [ ] Solution specific (not vague; could implement from this)
- [ ] Timeline realistic (not optimistic; add 20% buffer)
- [ ] Budget fully costed (no hidden expenses)
- [ ] ROI justified (revenue or cost savings > cost)
- [ ] Risks acknowledged (not hidden)
- [ ] One-page executive summary included
- [ ] Visual diagram of solution included
- [ ] Prereq links provided (designs, specs, relevant PRs)
- [ ] Approval mechanism clear (reply LGTM, sign here, etc.)
- [ ] Reviewer background documented (why ask them?)
```

**Reviewer Selection Strategy**
Don't submit to "everyone." Identify the single decision-maker:

```
For engineering proposals:
→ Go to CTO or engineering lead first
→ Only CC executive after tech approval

For product proposals:
→ Go to product lead first
→ Then loop in business stakeholder

For business/ops proposals:
→ Go to CFO or operations lead
→ Only executive approval if budget >$50K
```

Fewer reviewers = faster decisions. More cooks spoil the approval.

**Response Follow-Up Timing**
- Day 1: Send proposal (include review deadline, suggest 3-5 days)
- Day 4: (If no response) Ping with "checking in on timeline"
- Day 6: (If still no response) Offer to sync 15-min call vs. more questions needed
- Day 7: Escalate if critical path item blocked

## Troubleshooting

**Configuration changes not taking effect**

Restart the relevant service or application after making changes. Some settings require a full system reboot. Verify the configuration file path is correct and the syntax is valid.

**Permission denied errors**

Run the command with `sudo` for system-level operations, or check that your user account has the necessary permissions. On macOS, you may need to grant terminal access in System Settings > Privacy & Security.

**Connection or network-related failures**

Check your internet connection and firewall settings. If using a VPN, try disconnecting temporarily to isolate the issue. Verify that the target server or service is accessible from your network.


## Frequently Asked Questions

**How long does it take to write async project proposals that get approved?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Can I adapt this for a different tech stack?**

Yes, the underlying concepts transfer to other stacks, though the specific implementation details will differ. Look for equivalent libraries and patterns in your target stack. The architecture and workflow design remain similar even when the syntax changes.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [How to Write Async Technical RFCs That Get Meaningful](/remote-work-tools/how-to-write-async-technical-rfcs-that-get-meaningful-feedba/)
- [How to Write Clear Async Project Briefs for Remote Teams](/remote-work-tools/how-to-write-clear-async-project-briefs-for-remote-teams-avo/)
- [How to Write Freelance Proposals That Win](/remote-work-tools/how-to-write-freelance-proposals-that-win/)
- [How to Communicate Project Delays Remotely to Stakeholders](/remote-work-tools/how-to-communicate-project-delays-remotely-to-stakeholders-w/)
- [How to Write Async Daily Logs That Help Future Team Members](/remote-work-tools/how-to-write-async-daily-logs-that-help-future-team-members/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
