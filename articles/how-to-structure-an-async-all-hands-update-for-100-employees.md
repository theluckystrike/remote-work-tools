---
layout: default
title: "How to Structure an Async All Hands Update for 100 Employees"
description: "A practical guide to running asynchronous all hands meetings at scale. Templates, tools, and code examples for 100-person teams"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /how-to-structure-an-async-all-hands-update-for-100-employees/
categories: [guides]
reviewed: true
score: 9
intent-checked: true
voice-checked: true
tags: [remote-work-tools]---


{% raw %}

Structure your async all-hands around five consistent sections (company overview, department highlights, recognition, upcoming events, and Q&A), automate collection from department heads with a deadline-driven script, and distribute on the same weekday each month with a clear read-acknowledgment call-to-action. This format replaces the scheduling nightmare of synchronous all-hands for 100 employees while keeping engagement measurable through view counts, question volume, and acknowledgment rates. Below is the full step-by-step system including templates, automation code, and common pitfalls to avoid.

## Why Async All-Hands Works at Scale

When your team spans multiple time zones, finding a single hour that works for everyone becomes mathematically impossible. A 100-person team likely spans 8+ hour time differences, making synchronous all-hands either exclusionary or exhausting (or both).

Async updates respect individual work rhythms. Team members consume the update when they're focused, during their peak productivity hours. This flexibility increases the likelihood they'll actually absorb the information rather than half-listening while waiting for the portion that affects them.

The challenge shifts from scheduling to structure. Without the constraint of real-time attention, you must create content compelling enough to hold interest and organized enough to navigate quickly.

## Step 1: Define Your Update Sections

Every async all-hands update needs consistent sections your team learns to expect. Consistency reduces cognitive load—readers know where to find what they need.

Structure your update with these five components:

1. **Company Overview** — High-level metrics, major wins, current priorities
2. **Department Highlights** — Brief updates from each major team (engineering, product, sales, operations)
3. **Employee Recognition** — Kudos, promotions, anniversaries
4. **Upcoming Events** — Deadlines, holidays, planned communications
5. **Q&A Thread** — Collected questions with answers, or clarification on ambiguity

Keep each section under 200 words. At 100 employees, you have limited attention budget—brevity signals respect for their time.

## Step 2: Use a Template System

Create a reusable template that authors fill in consistently. This ensures nothing gets missed and makes comparison across updates easy.

Here's a practical template structure in markdown:

```markdown
# Company Update — [Month Year]

## Company Overview
<!-- 150-200 words: metrics, priorities, strategic direction -->

## Department Highlights
### Engineering
<!-- 100 words max -->

### Product
<!-- 100 words max -->

### Sales
<!-- 100 words max -->

### Operations
<!-- 100 words max -->

## Recognition
<!-- Promotions, kudos, tenure -->

## Coming Up
<!-- Next 4-6 weeks: deadlines, events, dates -->

## Q&A
<!-- Pre-submitted questions with answers -->
```

Distribute this template to department heads at least 5 business days before the update deadline. This lead time prevents rushed, low-quality submissions.

## Step 3: Automate Collection and Formatting

Manually collecting updates from 5-7 departments becomes a coordination nightmare at scale. Build a lightweight automation pipeline using familiar tools.

Create a shared folder (Google Drive, Notion, or GitHub) where department heads add their sections by a deadline. Use a simple script to compile these into the final document:

```python
#!/usr/bin/env python3
"""Compile async all-hands update from department files."""

import os
from pathlib import Path

SECTIONS = [
    "company_overview.md",
    "engineering.md",
    "product.md",
    "sales.md",
    "operations.md",
    "recognition.md",
    "coming_up.md"
]

def compile_update(source_dir: str, output_file: str):
    source = Path(source_dir)
    output = Path(output_file)

    content = ["# Company Update\n"]

    for section_file in SECTIONS:
        section_path = source / section_file
        if section_path.exists():
            content.append(f"\n## {section_file.replace('.md', '').replace('_', ' ').title()}\n")
            content.append(section_path.read_text())

    output.write_text('\n'.join(content))
    print(f"Update compiled: {output}")

if __name__ == "__main__":
    compile_update("updates/march-2026", "compiled-update.md")
```

This approach scales to any number of departments without additional manual effort. Each department owns their filename, the script assembles the final document.

## Step 4: Time Your Distribution Strategically

The timing of your async all-hands significantly impacts engagement. Send updates at the start of a work week (Tuesday or Wednesday) to avoid Monday backlog and Friday wind-down.

Choose a consistent day each month. Team members internalize the rhythm and check for updates automatically. Pattern recognition increases the likelihood they'll actually read it.

Pair the update with a clear call-to-action: "Please review and submit questions by Thursday" or "React with 👍 if you've read this." Simple engagement triggers boost completion rates without adding friction.

## Step 5: Handle Questions Asynchronously

The Q&A section distinguishes a true async all-hands from an one-way broadcast. Collect questions in advance through a simple form (Google Forms, Typeform, or a dedicated Slack channel).

Process questions in two ways:

- **Common questions** — Answer directly in the written update with full context
- **Complex or sensitive questions** — Note that leadership will address in the next synchronous meeting, or schedule a brief async follow-up

This hybrid approach gives you the benefits of async (thoughtful responses, no scheduling nightmares) while still handling issues that require live discussion.

## Measuring Engagement

Without real-time attendance, you need alternative metrics:

- **View count** — Track document opens or page views
- **Question volume** — More questions indicate investment
- **Reaction/acknowledgment** — Simple emoji responses show people read it
- **Follow-up clarity** — Track how many people need clarification on items covered

Review these metrics monthly. If engagement drops, adjust length, timing, or structure. The goal is continuous improvement, not rigid adherence to a fixed format.

## Common Pitfalls to Avoid

**Too long** — Updates exceeding 1,000 words see sharply declining engagement. Edit ruthlessly.

**No accountability** — Without explicit "acknowledged" actions, people assume someone else will handle it. Include specific asks: "Engineering team: review the Q3 roadmap by Friday."

**Inconsistent timing** — Erratic schedules cause people to stop checking. Commit to a predictable cadence regardless of how much there is to communicate.

**One-directional** — If the update never incorporates employee input, it becomes company broadcast, not company communication. The Q&A section must contain real responses to real questions.

## Tools That Support Async All-Hands

While the process matters more than the tool, certain platforms improve execution:

- **Notion** — Collaborative editing with database views for tracking
- **GitHub** — Markdown-based workflow with version control
- **Slack** — Threaded discussions for Q&A follow-up
- **Loom** — Optional video updates for personal touch (embed in document)

Choose tools your team already uses. Introducing new platforms for all-hands creates adoption friction that undermines the async goal.

## Scaling the System as Your Company Grows

The five-section structure scales beyond 100 employees by adding management layers. At 150+ employees, department-level overviews are replaced with manager-led section owners who consolidate team feedback. Here's how to extend the system:

### Adding Tier-Two Aggregation

Instead of collecting from 7 departments directly, designate a tier-two owner for each major functional area who collects input from their sub-teams:

```yaml
# Structure for 150-person organization
Company Overview: CEO (writes directly)
Engineering:
  - Platform Team Lead (collects from 5 engineers)
  - Product Team Lead (collects from 4 engineers)
  - Infrastructure Team Lead (collects from 3 engineers)
Product: VP Product (collects from 3 product managers)
Sales: VP Sales (collects from sales & customer success)
Operations: COO (collects from finance, legal, HR)
```

This prevents information bottlenecks while keeping the update focused and readable.

### Handling Time Zone Distribution

For globally distributed 100-person teams, declare one time zone as the "update standard" and accept that some people will read it outside their business hours. The asynchronous nature actually handles this better than synchronous meetings would—team members control when they consume information.

If your company spans 8+ hours across time zones, consider translating the core company overview into 2-3 languages. This increases accessibility and signals that the company values inclusive communication.

### Tracking Acknowledgment at Scale

With 100 people, simple emoji reactions create notification fatigue. Instead, create a Google Form that captures:

1. Name (optional, for anonymity if preferred)
2. "I've reviewed the all-hands" checkbox
3. Key question or clarification needed
4. Department they're in

This provides aggregate metrics—what percentage of each department read the update—while generating a list of questions that automatically feeds into the Q&A section next month.

```python
# Simple script to process acknowledgment form responses
import gspread
from collections import defaultdict

def analyze_update_engagement(form_responses_sheet):
    """Analyze engagement by department."""
    responses = form_responses_sheet.get_all_records()

    department_stats = defaultdict(lambda: {"read": 0, "total": 0})
    unanswered_questions = []

    for response in responses:
        dept = response['department']
        department_stats[dept]['total'] += 1

        if response['acknowledged']:
            department_stats[dept]['read'] += 1

        if response['question']:
            unanswered_questions.append({
                'question': response['question'],
                'department': dept
            })

    # Generate report
    print("Read Rates by Department:")
    for dept, stats in department_stats.items():
        rate = (stats['read'] / stats['total'] * 100)
        print(f"{dept}: {rate:.1f}% ({stats['read']}/{stats['total']})")

    return unanswered_questions
```

## Common Questions About Scaling

**Q: What happens if critical decisions come up mid-month that can't wait?**
A: Use an emergency all-hands supplement. Keep it brief (under 200 words) and explicitly mark it as urgent. Don't force it into the regular monthly structure—this preserves the monthly rhythm's reliability.

**Q: How do we handle real-time crisis communication (security incident, major outage)?**
A: Async all-hands replace routine status updates, not crisis communication. Keep a separate incident communication channel and response template. After the crisis resolves, include a retrospective in the next scheduled all-hands.

**Q: Should we do video updates for all 100 employees?**
A: Optional. A 5-minute video from the CEO introducing the monthly update can add personal touch without disrupting the async format. Keep it optional—not everyone prefers video-first communication, and transcripts help those who are deaf or hard of hearing.

**Q: How do we prevent the all-hands from becoming a broadcast with no real dialogue?**
A: The Q&A section is non-negotiable. Even if you receive only three questions, answer them thoroughly. This signals that you value employee input. If you notice declining questions, explicitly ask: "What do you want to know about our direction?" in your next all-hands.
---

An async all-hands for 100 employees succeeds through structure, not magic. Define clear sections, automate collection, time distribution consistently, and close the loop with real Q&A. Your team gets information they can actually absorb, and you get a scalable communication system that works regardless of team size or time zone distribution.

## Frequently Asked Questions

**How long does it take to structure an async all hands update for 100 employees?**

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

- [Example: project-update.yml - Scheduled updates structure](/remote-work-tools/how-to-manage-client-expectations-when-team-works-asynchrono/)
- [Veed API - Upload and process video](/remote-work-tools/remote-team-async-video-update-tool-comparison-loom-vs-veed-/)
- [Async Mentorship Program Structure for Remote Junior Develop](/remote-work-tools/async-mentorship-program-structure-for-remote-junior-develop/)
- [Best Hot Desking Software for Hybrid Offices with Under 100](/remote-work-tools/best-hot-desking-software-for-hybrid-offices-with-under-100-employees-2026/)
- [Best Practice for Hybrid Team All Hands Meeting with Mixed](/remote-work-tools/best-practice-for-hybrid-team-all-hands-meeting-with-mixed-i/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

