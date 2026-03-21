---
layout: default
title: "How to Run Async Sprint Demos with Recorded Walkthroughs"
description: "Learn practical methods for recording sprint demos asynchronously. This guide covers tools, workflows, and code snippets for effective async presentations"
date: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /how-to-run-async-sprint-demos-with-recorded-walkthroughs-for/
categories: [guides]
tags: [remote-work-tools, sprint-demos, async-communication, remote-work, stakeholders]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---


{% raw %}
# How to Run Async Sprint Demos with Recorded Walkthroughs for Stakeholders

Record a screen walkthrough under 10 minutes following a consistent structure -- 30-second sprint overview, 2-5 minutes per feature demo, optional technical highlights, and 30-second next-steps summary -- then distribute it with timestamps and a written summary so stakeholders can review on their own schedule. This eliminates the time zone conflicts of live demos while creating a permanent searchable record of sprint progress.

## Why Async Demos Work Better for Distributed Teams

Traditional sprint demos force everyone into a single meeting time, often meaning someone joins at 7 AM or 10 PM. Async recordings eliminate this constraint entirely. Stakeholders can watch during their productive hours, pause to review complex sections, and revisit recordings later when questions arise.

The key benefits include:

- **Time zone flexibility** — no one needs to attend live
- **Playback control** — stakeholders speed up or rewatch sections
- **Permanent documentation** — recordings serve as historical records
- **Reduced meeting fatigue** — async communication respects everyone's calendar

## Recording Your Sprint Demo

### Option 1: CLI-Based Screen Recording with ffmpeg

For developers who prefer command-line tools, you can automate screen recording using ffmpeg. This approach works well for consistent, repeatable demo recording.

Install ffmpeg first:

```bash
# macOS
brew install ffmpeg

# Ubuntu/Debian
sudo apt install ffmpeg
```

Create a recording script:

```bash
#!/bin/bash
# record-demo.sh

OUTPUT_DIR="./sprint-recordings"
DATE=$(date +%Y-%m-%d)
OUTPUT_FILE="$OUTPUT_DIR/sprint-demo-$DATE.mp4"

# Capture screen at 1080p, 30fps
ffmpeg -f avfoundation -i "1:0" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  -s 1920x1080 -r 30 \
  "$OUTPUT_FILE"
```

Run the script to start recording. Press `q` to stop when finished.

### Option 2: Native Screen Recording Tools

Most operating systems include built-in screen recording:

- macOS: Use Command+Shift+5 to open the screenshot toolbar, then select screen recording
- Windows: Press Windows+Alt+R or use the Xbox Game Bar
- Linux: Use GNOME's built-in recorder (Alt+Ctrl+Shift+R) or SimpleScreenRecorder

These tools are easier for quick demos and require no setup.

## Structuring Your Walkthrough

A good async demo walkthrough follows a consistent structure. Stakeholders should know what to expect and where to find key information.

### Recommended Demo Structure

1. **Overview** (30 seconds)
 - Sprint goal and scope
 - What was completed vs. planned

2. **Feature Walkthrough** (2-5 minutes per feature)
 - Show the feature in action
 - Narrate what you're demonstrating
 - Highlight key decisions or tradeoffs

3. **Technical Highlights** (optional, 1-2 minutes)
 - Architecture changes
 - Performance improvements
 - Code refactoring

4. **Next Steps** (30 seconds)
 - What's coming in the next sprint
 - Dependencies or blockers

### Recording Best Practices

- **Speak clearly and narrate actions** — viewers can't ask questions in real-time
- **Highlight mouse movements** — use a tool like Keycastr on macOS to show keystrokes
- **Keep recordings under 10 minutes** — attention drops significantly longer
- **Show, don't just describe** — demonstrate the actual feature working

## Automating Demo Video Generation

For teams building CI/CD pipelines, you can automate demo video creation using tools like Capture It. Here's a GitHub Actions workflow that records test runs:

```yaml
name: Record Demo
on: [push]

jobs:
  record:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: brew install ffmpeg
      - name: Record screen
        run: |
          ffmpeg -f avfoundation -i "1:0" \
            -t 60 demo.mp4
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: sprint-demo
          path: demo.mp4
```

This records the first 60 seconds of activity and stores it as an artifact. Extend the duration or trigger on specific events for more targeted recordings.

## Distributing to Stakeholders

Once recorded, get the video to stakeholders effectively:

### Platform Options

- GitHub Issues: Upload videos as artifacts or link to hosted videos
- Slack: Share directly in channels, but be mindful of file size limits
- Notion/Confluence: Embed videos in team spaces for permanent storage
- YouTube (unlisted): Good for longer recordings with timestamps

### Add Context with Description

Always include a written summary with your video:

```markdown
## Sprint 24 Demo Recording

**Duration**: 8:32

**Features Shown**:
- User dashboard redesign
- New export functionality
- Performance improvements

**Timestamps**:
- 0:00 - Sprint overview
- 1:45 - Dashboard walkthrough
- 4:20 - Export feature
- 6:10 - Performance metrics

**Questions to review**: Please share feedback by Thursday EOD.
```

## Handling Feedback Async

The demo isn't complete until you've gathered feedback. Set up a clear async feedback loop:

1. Deadlines: Specify when stakeholders should review (e.g., "by Thursday")
2. Format: Ask for specific feedback (e.g., "approve" or "request changes")
3. Channel: Designate where to collect responses (GitHub issue, Slack thread)
4. Follow-up: Summarize feedback in your next standup or async update

## Tools Worth Considering

Several tools specialize in async presentations:

- Loom: Quick recordings with links and comments
- Vidyard: Business-focused with analytics
- Screen Studio: Simple, high-quality Mac screen recording
- OBS: Free, cross-platform, highly customizable

Choose based on your team's existing tools and workflow. The best tool is one your team will actually use consistently.

## Measuring Success

Track whether async demos are working for your team:

- Review completion rate: Are stakeholders watching?
- Feedback quality: Are you getting actionable responses?
- Time saved: Compare to synchronous demo meeting hours
- Stakeholder satisfaction: Quick pulse survey after each sprint

Iterate on your approach based on these metrics.

---

Running async sprint demos requires upfront investment in recording habits and workflows, but pays dividends in team flexibility and stakeholder engagement. Start with simple recordings, gather feedback, and refine your process over time.




## Related Articles

- [Async Pair Programming Workflow Using Recorded Walkthroughs](/remote-work-tools/async-pair-programming-workflow-using-recorded-walkthroughs-and-github/)
- [How to Present Sprint Demos to Non-Technical Remote Clients](/remote-work-tools/how-to-present-sprint-demos-to-non-technical-remote-clients/)
- [Async Product Discovery Process for Remote Teams Using](/remote-work-tools/async-product-discovery-process-for-remote-teams-using-recorded-interviews/)
- [Async Team Retrospective Using Shared Documents and](/remote-work-tools/async-team-retrospective-using-shared-documents-and-recorded/)
- [How to Do Async Code Pairing with Recorded Screen Share](/remote-work-tools/how-to-do-async-code-pairing-with-recorded-screen-share-sessions/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
