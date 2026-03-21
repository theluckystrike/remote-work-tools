---
layout: default
title: "Example: Export Miro board via API"
description: "Learn practical techniques for running effective remote workshops in Miro with structured communication exercises that keep teams engaged and productive"
date: 2026-03-16
author: "Remote Work Tools"
permalink: /how-to-help-remote-team-workshops-using-miro-with-stru/
reviewed: true
score: 9
intent-checked: true
voice-checked: true
categories: [guides]
tags: [remote-work-tools, remote-work]
---

{% raw %}

Run effective remote workshops in Miro by combining structured communication exercises like round-robin protocols, async brainstorming, parking lot management, dot voting, and breakout frames with clear time boundaries and organized visual layouts. These techniques ensure equal participation, maintain engagement, and drive actionable outcomes for distributed teams.

Running productive remote workshops presents unique challenges. Without the benefit of physical presence, facilitators must work harder to maintain engagement, ensure equal participation, and drive meaningful outcomes. Miro provides a powerful collaborative canvas, but the tool alone doesn't guarantee successful workshops. Combining Miro's features with structured communication exercises creates a framework that transforms async collaboration into focused,高效 sessions.

This guide covers practical techniques for helping remote team workshops using Miro, with emphasis on structured communication exercises that developers and power users can implement immediately.

## Setting Up Your Miro Workshop Environment

Before starting any workshop, prepare your canvas with clear sections. Create distinct zones for different phases of your session:

```
┌─────────────────────────────────────────────────────────┐
│  WORKSHOP: [Session Title]                              │
├──────────────┬──────────────┬──────────────┬────────────┤
│  WARM-UP     │  MAIN        │  BREAKOUT    │  SYNTHESIS │
│  (5 min)     │  ACTIVITY    │  DISCUSSIONS │  (10 min)  │
│              │  (20 min)    │  (15 min)    │            │
└──────────────┴──────────────┴──────────────┴────────────┘
```

Use color-coded sticky notes to differentiate participant inputs. Assign specific colors to team members or input types (questions, ideas, blockers). This visual organization helps participants quickly scan the canvas and find relevant content.

## Structured Communication Exercise: The Round-Robin Protocol

One of the most effective techniques for remote workshops is the round-robin protocol. This ensures every participant has equal speaking time and reduces the dominance of vocal team members.

### Implementation Steps

1. **Create a timer frame** in Miro using the timer widget or a simple sticky note with a countdown
2. **Assign speaking order** using numbered sticky notes arranged in a circle
3. Set explicit rules: Each person speaks for exactly 2 minutes when their turn arrives
4. **Use the "pass" option** for participants who want to skip their turn

For developers, this structure works well during code review discussions, architecture planning, and incident post-mortems. The fixed time allocation prevents discussions from spiraling while ensuring all perspectives get heard.

## Icebreaker Exercise: Async Brainstorm Mapping

For distributed teams spanning multiple time zones, async workshops require different help approaches. Use this structured exercise to gather input before synchronous sessions:

### Setup Template

```javascript
// Miro API: Create async input frame programmatically
const workshopFrame = {
  title: "Feature Brainstorm - Week 12",
  sections: [
    { id: "opportunities", label: "Opportunities", color: "#4ADE80" },
    { id: "constraints", label: "Technical Constraints", color: "#F87171" },
    { id: "questions", label: "Open Questions", color: "#60A5FA" }
  ],
  deadline: "2026-03-20T18:00:00Z"
};
```

Participants add sticky notes to appropriate sections before the sync meeting. During the live session, the facilitator reviews patterns and clusters similar ideas together using Miro's grouping feature.

## The停车场 (Parking Lot) Technique

Remote workshops often generate tangents—valuable discussions that deserve attention but fall outside the session's scope. Create a dedicated "Parking Lot" section on your canvas:

- Add a frame labeled "Parking Lot" in the corner
- When tangential topics arise, move a sticky note to this area
- Assign a follow-up owner for each parked item
- Review parking lot items at session end or in subsequent meetings

This technique maintains session focus while validating contributions that warrant future discussion.

## Real-Time Collaboration: Voting and Prioritization

Miro's voting features enable democratic decision-making in real-time workshops. Use dot voting for prioritizing features, selecting approaches, or identifying the most important blockers.

### Dot Voting Workflow

1. Present options as cards or sticky notes in a central area
2. Explain voting rules: each participant gets 3-5 dots
3. Enable anonymous voting in Miro's voting settings
4. Allow 2-3 minutes for voting
5. Sort results by vote count automatically

For engineering teams, this works exceptionally well for tech debt prioritization, RFC review, and sprint planning. The visual result immediately shows team consensus without lengthy debate.

## Breakout Exercise: Pair Mapping

For complex problems, divide participants into smaller groups for focused discussion. Miro's breakout frames feature allows simultaneous collaboration in separate canvas sections.

### Help Template

```
┌─────────────────────────────────────────┐
│  MAIN CANVAS                           │
│  [Instructions here]                    │
├─────────────┬───────────────────────────┤
│  GROUP A    │  GROUP B                  │
│  [Frame 1]  │  [Frame 2]                │
│             │                           │
├─────────────┼───────────────────────────┤
│  GROUP C    │  GROUP D                  │
│  [Frame 3]  │  [Frame 4]                │
│             │                           │
└─────────────┴───────────────────────────┘
```

After breakout sessions, reconvene and have each group present their findings. Use the timer widget to enforce strict time limits per group.

## Documentation and Follow-Up

The value of a well-helped workshop diminishes without proper documentation. After each session:

1. Export the canvas as PDF for permanent record
2. Screenshot key decision points
3. Create action items in your project management tool
4. Share summary within 24 hours

```bash
# Example: Export Miro board via API
curl -X POST "https://api.miro.com/v2/boards/{board_id}/export" \
  -H "Authorization: Bearer $MIRO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"format": "pdf", "quality": "high"}'
```

## Alternative Tools and Comparison

While Miro dominates collaborative whiteboarding, several alternatives offer distinct advantages:

**Miro ($10-16/month per user):**
- Strengths: Largest library of templates, strong API, excellent performance with large groups
- Best for: Teams already using Miro, developers wanting API integration
- Limitations: Cost adds up at scale, requires subscription commitment

**FigJam (included in Figma Professional - $12/month):**
- Strengths: Deep integration with design tools, multiplayer editing, real-time communication
- Best for: Design-heavy teams, teams using Figma already
- Limitations: Less whiteboarding-specific than Miro, smaller template library

**Mural ($12/month):**
- Strengths: Strong facilitation tools, built-in timers, dedicated workshop templates
- Best for: Facilitators running many workshops, teams needing meeting structure
- Limitations: Higher learning curve, fewer integrations than Miro

**Google Jamboard (Free with Google Workspace):**
- Strengths: Simple, familiar interface, no additional cost if using Google Workspace
- Best for: Budget-conscious teams, Google Workspace users, simple diagrams
- Limitations: Limited advanced features, fewer collaborative tools

**Excalidraw (Free):**
- Strengths: Lightweight, open-source, no account required
- Best for: Quick collaborative sketching, developers preferring open tools
- Limitations: Minimal structure/templates, fewer facilitation features

**Pricing Comparison Summary:**
- Miro: $10-16/month per user, ~$120-192/year individual
- FigJam: Included with Figma Professional ($12/month), ~$144/year
- Mural: $12/month, ~$144/year
- Google Jamboard: Free with Google Workspace ($6-18/month depending on tier)
- Excalidraw: Free (open-source, self-hosted option available)

For small teams (5-10 people), FigJam or free options often provide better ROI than dedicated whiteboarding software. For larger teams or heavy workshop users, Miro's investment pays off through time saved and better outcomes.

## Facilitator Checklists for Different Workshop Types

**Architecture Review Workshop (90 minutes):**

1. **Pre-workshop (1 week):** Distribute architecture diagram for async review
2. **Opening (5 min):** Clarify decision scope and constraints
3. **Async input (15 min):** Participants add questions/concerns to designated frame
4. **Presentation (20 min):** Architect walks through design decisions
5. **Guided critique (30 min):** Round-robin protocol for structured feedback
6. **Parking lot review (10 min):** Address tangential issues identified
7. **Action items (5 min):** Assign follow-ups and next meeting
8. **Post-workshop:** Export decision record within 24 hours

**Feature Brainstorm Workshop (60 minutes):**

1. **Warm-up (5 min):** Share recent successful features
2. **Async input (ongoing):** Team members add ideas during week before meeting
3. **Clustering (15 min):** Group related ideas, identify themes
4. **Impact/effort analysis (15 min):** Dot voting to score each cluster
5. **Discussion (15 min):** Talk through top-voted items
6. **Next steps (5 min):** Decide which ideas to prototype/explore
7. **Post-workshop:** Create tickets from validated ideas

**Retrospective Workshop (60 minutes):**

1. **Safe space (2 min):** Remind team of psychological safety commitment
2. **Async input (20 min during week):** Team members add what went well/could improve
3. **Grouping (10 min):** Cluster related feedback
4. **Discussion (15 min):** Talk through top themes
5. **Action planning (10 min):** Pick 1-2 experiments to try
6. **Commitment (3 min):** Team agrees on what to measure
7. **Post-workshop:** Track follow-ups in dedicated retro document

## Technical Setup for Large Group Workshops (20+ participants)

Miro performance degrades with many simultaneous editors. For larger groups:

1. **Use breakout boards:** Divide into groups of 5-8, each in separate frames
2. **Assign facilitators:** One person manages each breakout group
3. **Stagger editing:** Have groups work sequentially rather than simultaneously
4. **Simplify visuals:** Reduce complexity of main board when over 15 participants
5. **Use voting instead of live editing:** More participants = more voting features, fewer live edits

```bash
#!/bin/bash
# Miro performance tuning for large workshops
# Run before major facilitation

echo "Optimizing Miro board for large group..."

# Archive old prototype boards (unused content slows performance)
# Delete duplicate frames
# Reduce image resolution (replace high-res images with optimized versions)
# Simplify complex nested frame structures
# Test performance in private test meeting before launching with full group

echo "Miro optimization complete"
```

## Documentation and Institutional Memory

A workshop's value extends far beyond the 90 minutes if properly documented:

**During workshop:**
- Assign note-taker to capture decisions and action items in parallel
- Take screenshots at key points
- Record consent for video recording

**Immediately after (within 2 hours):**
- Export board as PDF with high resolution
- Create summary document with decisions and next steps
- Assign owners to action items with due dates

**Within 24 hours:**
- Share summary with team
- Create tickets for action items
- Archive board with clear naming convention

**Ongoing:**
- Link workshop outputs to related project documentation
- Reference in future related workshops
- Update outcomes 30-60 days post-workshop to show what actually happened with recommendations

This creates a searchable archive that new team members can review to understand how decisions were made and why current practices exist.

---



## Related Articles

- [How to Run Effective Remote Client Workshops Using Miro](/remote-work-tools/how-to-run-effective-remote-client-workshops-using-miro-board/)
- [Example: Create a booking via API](/remote-work-tools/best-client-scheduling-tool-for-remote-agency-multiple-time-/)
- [Example: Trigger BambooHR onboarding workflow via API](/remote-work-tools/best-onboarding-platform-for-remote-companies-processing-mor/)
- [Example: Verify MFA is enabled via API (GitHub Enterprise)](/remote-work-tools/how-to-create-security-onboarding-checklist-for-new-remote-t/)
- [How to Create a Remote Team Values Wall Using Miro Board](/remote-work-tools/how-to-create-remote-team-values-wall-using-miro-board/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
