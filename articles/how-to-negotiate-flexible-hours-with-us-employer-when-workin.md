---
layout: default
title: "Example on-call schedule that uses timezone difference"
description: "A practical guide for developers in Europe working with US companies. Learn negotiation strategies, overlap calculations, and async workflows to secure"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools"
permalink: /how-to-negotiate-flexible-hours-with-us-employer-when-workin/
reviewed: true
score: 9
voice-checked: true
categories: [guides]
intent-checked: true
tags: [remote-work-tools]---
---
layout: default
title: "Example on-call schedule that uses timezone difference"
description: "A practical guide for developers in Europe working with US companies. Learn negotiation strategies, overlap calculations, and async workflows to secure"
date: 2026-03-16
last_modified_at: 2026-03-16
author: "Remote Work Tools"
permalink: /how-to-negotiate-flexible-hours-with-us-employer-when-workin/
reviewed: true
score: 9
voice-checked: true
categories: [guides]
intent-checked: true
tags: [remote-work-tools]---

Propose specific alternatives like 10 AM - 4 PM CET core hours (overlapping 2-4 PM US East Coast), showing how this gives the US team morning hours for meetings while you work during peak productivity. Demonstrate your async capability with PR descriptions, async video walkthroughs, and 24-hour code review turnaround for two weeks before the negotiation, then present this track record as proof that flexible hours don't mean unavailability. If denied initially, start with 1-2 flexible days weekly as a trial, document your productivity metrics, then revisit the conversation once you've proven the arrangement works.

## Understanding the Timezone Math

The first step is knowing exactly what you're working with. US timezones span Eastern (UTC-5), Central (UTC-6), Mountain (UTC-7), and Pacific (UTC-8). If you're in Western Europe (UTC+0/+1), your overlap with US teams ranges from 1-4 hours during standard business hours.

Here's a quick calculation to understand your overlap:

```javascript
// Calculate timezone overlap between European and US locations
function calculateOverlap(europeOffset, usOffset, europeStartHour = 9, europeEndHour = 18) {
  // Convert to 24-hour format for easier math
  const europeWorkStart = europeStartHour - europeOffset;
  const europeWorkEnd = europeEndHour - europeOffset;
  const usWorkStart = 9 - usOffset;  // US 9 AM
  const usWorkEnd = 17 - usOffset;   // US 5 PM

  const overlapStart = Math.max(europeWorkStart, usWorkStart);
  const overlapEnd = Math.min(europeWorkEnd, usWorkEnd);

  return {
    hours: Math.max(0, overlapEnd - overlapStart),
    start: overlapStart,
    end: overlapEnd
  };
}

// Example: Berlin (UTC+1) vs New York (UTC-5)
const berlinNY = calculateOverlap(1, -5);
console.log(`Berlin-NYC overlap: ${berlinNY.hours} hours (${berlinNY.start}:00 to ${berlinNY.end}:00 UTC)`);

// Example: Lisbon (UTC+0) vs Los Angeles (UTC-8)
const lisbonLA = calculateOverlap(0, -8);
console.log(`Lisbon-LA overlap: ${lisbonLA.hours} hours (${lisbonLA.start}:00 to ${lisbonLA.end}:00 UTC)`);
```

This code reveals the hard truth: your overlap might be only 1-3 hours during traditional business hours. That's not enough time for real-time collaboration, which is exactly why flexible hours matter.

## Build Your Business Case

Before approaching your employer, prepare data that demonstrates how flexible hours actually improve your output. US managers often worry that non-standard hours mean unavailability. Counter this with concrete points:

**Document your current availability patterns.** Track your most productive hours for two weeks. If you're a backend developer who codes best at 6 AM local time, that's valuable information. Early morning hours often align well with late afternoon US time.

**Show how async communication already works.** Prove that you can deliver without real-time check-ins. Submit pull requests with thorough descriptions, write detailed status updates, and maintain clear documentation. When your work speaks for itself, managers gain confidence in flexible arrangements.

**Calculate the business value.** If you're in Portugal negotiating with a San Francisco team, your 8 AM local time is 11 PM PST the previous day. But your 10 AM local time is 2 AM PST—useless for collaboration. However, your 2 PM local time is 6 AM PST, perfect for catching the US team as they start their day. Strategic hour alignment can actually expand effective collaboration windows.

## Propose Specific Alternatives

Vague requests get vague answers. Come with concrete proposals:

### Option 1: Split Core Hours

```text
My Proposal: 10:00 - 15:00 CET (1:00 - 6:00 AM PST)
Core availability: 10:00 - 15:00 CET for real-time meetings
Async coverage: Extended hours for code reviews and PR feedback
US team overlap: 1 PM - 3 PM CET (6 AM - 8 AM PST)
```

This gives the US team morning hours when they're fresh, while you work during your most productive afternoon block.

### Option 2: Asynchronous-First Model

Propose that all non-urgent communication happens asynchronously, with designated overlap windows only for critical sync:

```javascript
// Define your async-friendly response expectations
const availability = {
  timezone: 'CET',
  coreHours: '10:00 - 16:00',
  asyncResponseTime: '24 hours',
  urgentResponseTime: '2 hours',
  overlapWindows: [
    { day: 'Mon-Thu', local: '14:00-16:00', utc: '13:00-15:00', pstequivalent: '05:00-07:00' }
  ]
};
```

### Option 3: Staggered Start

Request starting earlier or later to maximize overlap. A 7 AM start in Berlin gives you 5 hours of overlap with New York:

| Berlin Time | NYC Time | Activity |
|------------|----------|----------|
| 7:00 - 9:00 | 1:00 - 3:00 | Deep work, code |
| 9:00 - 10:00 | 3:00 - 4:00 | Email, async communication |
| 10:00 - 12:00 | 4:00 - 6:00 | US team joins, collaboration |
| 12:00 - 14:00 | 6:00 - 8:00 | Meetings, reviews |
| 14:00 - 16:00 | 8:00 - 10:00 | Async documentation |

## Address Common Employer Concerns

**"What about emergencies?"** Establish an on-call rotation that accounts for timezone coverage. If you're in Europe and the US team is in California, you naturally cover different coverage windows:

```yaml
# Example on-call schedule that applies timezone difference
on_call_coverage:
  europe_team:
    timezone: CET
    hours: "16:00 - 24:00 UTC"
    covers: "US evening / after hours"
  us_team:
    timezone: PST
    hours: "16:00 - 24:00 UTC"
    covers: "EU evening / after hours"
```

**"How will we have meetings?"** Limit synchronous meetings to truly necessary ones. Use tools like World Time Buddy or When2meet to find optimal slots. Document decisions asynchronously to reduce meeting dependency.

**"Clients won't understand."** If your company serves US clients, position yourself as covering European timezone support—an asset rather than an obstacle. Document your hours clearly in client-facing materials.

## Present Your Proposal Professionally

Structure your request like a professional proposal:

1. State the problem: "The 6-hour timezone difference between my location (CET) and our main US office limits real-time collaboration to ~2 hours daily."

2. Propose the solution: "I'd like to adjust my core hours to 10 AM - 4 PM CET, with availability for meetings during our overlap window."

3. Show the benefits: "This gives us 4 hours of overlap, improves my code review turnaround, and maintains full team coverage across timezones."

4. Suggest a trial period: "I'm happy to try this for one sprint (2 weeks) and we can evaluate effectiveness."

5. Define success metrics: "We can track PR review time, meeting attendance, and delivery predictability during the trial."

## use Async Tools to Support Your Case

Show your employer that flexible hours work by demonstrating async communication competence:

- Write detailed PR descriptions instead of relying on in-person walkthroughs
- Use Loom or Screen Studio for video explanations instead of live demos
- Maintain a living document of decisions and context
- Respond to messages within committed timeframes consistently

When your manager sees you deliver reliably without requiring real-time availability, they become more open to formalizing flexible arrangements.

## What If Your Request Is Denied?

If initial negotiations don't succeed, explore alternatives:

- Gradual adjustment: Start with one or two flexible days per week
- Role-specific solutions: Some roles (backend, DevOps) naturally work better with async workflows
- Team-by-team approach: Perhaps one team member can accommodate your hours even if management won't formally change policy
- Document everything: Track your productivity during any informal flexibility you already exercise

Many developers have secured flexible hours by proving their value first and negotiating second. The key is demonstrating that your output quality remains high—or improves—when you're not forced to work during your biological trough hours.

## Comparative Schedules: Europe to US Timezone Mapping

Understanding exactly what overlap you get is critical. Here's a reference table showing real-world overlaps for common European and US city pairs:

| European City | US City | Peak Overlap | Hours |
|--------------|---------|-------------|-------|
| **Lisbon (UTC+0)** | San Francisco (UTC-8) | 5 PM-11 PM PST | 6 hours |
| **Berlin (UTC+1)** | New York (UTC-5) | 9 AM-1 PM EST | 4 hours |
| **Amsterdam (UTC+1)** | Chicago (UTC-6) | 10 AM-2 PM CST | 4 hours |
| **London (UTC+0)** | Los Angeles (UTC-8) | 5 PM-11 PM PST | 6 hours |
| **Paris (UTC+1)** | Boston (UTC-5) | 9 AM-1 PM EST | 4 hours |
| **Warsaw (UTC+1)** | Austin (UTC-6) | 10 AM-2 PM CST | 4 hours |

Use these natural overlap windows for synchronous work. Schedule meetings during peak overlap hours and protect them fiercely—they're your real-time collaboration time with the US team.

## Building Your Productivity Evidence Document

Before proposing flexible hours, compile a dossier that proves your capability to work independently. This document becomes your negotiation toolkit:

```markdown
## Async Capability Evidence (Current Month Example)

### Code Quality Metrics
- Average PR review cycle: 18 hours (down from 24 hours last quarter)
- First-pass approval rate: 87% (company average: 72%)
- Bugs found in PR reviews: 12 (company average: 8)

### Communication Performance
- Async message response time: 6.2 hours average
- Detailed PR descriptions: 100% of commits include 3+ line description
- Follow-up communication clarity: "Clear and detailed" rating from 5 peer reviews

### Delivery Consistency
- On-time delivery: 100% (last 8 sprints)
- Sprint velocity: 34 story points average (consistent)
- No missed deadlines or delayed deliverables

### Documentation Examples
- Maintained a running technical decision log for team
- Created debugging guide for complex subsystem (now referenced by 3 teammates)
- Wrote deployment runbook that reduced incident resolution time by 40%
```

Gather specific evidence over 2-3 weeks before your negotiation conversation. This document should be concrete and measurable, not opinion-based.

## The Negotiation Conversation Script

When you sit down (or video call) with your manager, have a structured conversation:

**Opening**: "I'd like to discuss my working hours. I've been thinking about timezone optimization and productivity, and I have a proposal that I think benefits both me and the team."

**The Ask**: "I'd like to adjust my core hours to [specific time range]. This maintains [X hours] of overlap with the US team during peak hours while letting me work during my most productive time."

**The Evidence**: Present your async capability document. "Here's what my last month of work looks like. I'm consistently delivering high-quality work with fast async communication. The flexible hours shouldn't change that."

**The Trials**: "I'd love to test this for one sprint—two weeks. We can track the same metrics: code review quality, communication response time, and delivery timeliness. If it's working, we make it permanent. If not, we adjust."

**The Boundaries**: "Here's what won't change: I'll be fully available for meetings during our overlap window, I'll maintain 12-hour async response time, and I'll document any blockers immediately."

## Handling the "Always On" Expectation

Some managers worry that flexible hours mean you'll be unavailable when they need you. Counter this explicitly by establishing clear boundaries:

```yaml
# Your Explicit Availability Framework
timezone: CET (Central European Time)

core_hours: 10:00 - 16:00 CET
  availability: 100% for meetings, Slack, decisions
  reason: Covers 2-8 AM PST (US early morning)

extended_async_hours: 08:00 - 10:00 & 16:00 - 18:00 CET
  availability: Check messages 2x daily, respond within 4 hours
  reason: Deep work time, but not completely unavailable

outside_hours: Before 08:00 or after 18:00 CET
  availability: No work expectation
  reason: Personal time, emergencies only

on_call_rotation:
  frequency: Once per month
  coverage: Extended hours for emergencies
  compensation: 4 extra hours of flexible time next week
```

This framework makes it clear you're not asking to disappear—you're asking to work during your best hours while maintaining better boundaries.

## If Your Negotiation Fails (Backup Strategies)

If your manager says no initially, you have several intermediate options:

**Incremental approach**: "Can I try flexible hours on Mondays and Fridays for two weeks?" Smaller changes face less resistance. After proving success, expand gradually.

**Project-based trial**: "For the [specific project] I'm leading, I'd like to try flexible hours. It'll help me focus on the complex architecture work without meetings." Tie it to specific business value.

**Output-based agreement**: "I'll maintain my current sprint velocity and code review quality. If those metrics drop, we revert to standard hours." Make it outcome-based, not time-based.

**Peer precedent**: "I noticed [other developer] has flexible hours. Would the same arrangement work for me?" If it's not a company policy issue, peer examples work.

**Regional support angle**: "My timezone naturally covers [specific coverage need] better than standard hours. Having me available 2 PM-8 PM UTC gives us better client support in the Asian region." Reframe it as a business advantage.

## Legal and HR Considerations

Before negotiating flexible hours, understand the legal market:

**Employment status**: Remote contractors have fewer protections than employees in many jurisdictions. If you're a contractor, this discussion happens at project negotiation, not during employment.

**Labor laws**: Some countries have strict regulations about working hours. Europe often requires documented agreements about flexible arrangements. Document any flexible hour agreement in writing.

**Visa/work permit requirements**: If you're in a country on a work visa, certain flexible hour arrangements might violate visa conditions. Check with immigration before changing hours officially.

**Tax implications**: Flexible hours might affect how your income is taxed or claimed. Unusual hour patterns shouldn't affect tax treatment, but document the arrangement in case of audit.

When in doubt, have your employer's HR department (not just your manager) acknowledge the flexible arrangement in writing.

## Frequently Asked Questions

**Who is this article written for?**

This article is written for developers, technical professionals, and power users who want practical guidance. Whether you are evaluating options or implementing a solution, the information here focuses on real-world applicability rather than theoretical overviews.

**How current is the information in this article?**

We update articles regularly to reflect the latest changes. However, tools and platforms evolve quickly. Always verify specific feature availability and pricing directly on the official website before making purchasing decisions.

**Are there free alternatives available?**

Free alternatives exist for most tool categories, though they typically come with limitations on features, usage volume, or support. Open-source options can fill some gaps if you are willing to handle setup and maintenance yourself. Evaluate whether the time savings from a paid tool justify the cost for your situation.

**How do I get started quickly?**

Pick one tool from the options discussed and sign up for a free trial. Spend 30 minutes on a real task from your daily work rather than running through tutorials. Real usage reveals fit faster than feature comparisons.

**What is the learning curve like?**

Most tools discussed here can be used productively within a few hours. Mastering advanced features takes 1-2 weeks of regular use. Focus on the 20% of features that cover 80% of your needs first, then explore advanced capabilities as specific needs arise.

## Related Articles

- [How to Schedule Meetings Across 8 Hour Timezone Difference](/remote-work-tools/how-to-schedule-meetings-across-8-hour-timezone-difference-w/)
- [Find overlapping work hours across three zones](/remote-work-tools/how-to-schedule-onboarding-meetings-across-time-zones-for-re/)
- [Team hours (as datetime.time objects converted to hours)](/remote-work-tools/how-to-calculate-timezone-overlap-hours-when-remote-team-spa/)
- [Example: EOR Integration Configuration](/remote-work-tools/best-employer-of-record-service-for-hiring-remote-developers/)
- [Example: Generating a staggered schedule for a 6-person team](/remote-work-tools/best-practice-for-hybrid-work-policy-covering-which-days-tea/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
