---
layout: default
title: "Virtual Meeting Etiquette Best Practices: A Developer Guide"
description: "Practical virtual meeting etiquette best practices for developers and power users. Includes technical tips, automation examples, and platform-specific"
date: 2026-03-15
last_modified_at: 2026-03-22
author: "Remote Work Tools Guide"
permalink: /virtual-meeting-etiquette-best-practices/
reviewed: true
score: 8
categories: [best-of]
intent-checked: true
voice-checked: true
tags: [remote-work-tools, best-of]
---


The three highest-impact virtual meeting practices are: test your audio and video before every call, mute when not speaking, and always review the agenda beforehand. These habits alone eliminate the most common meeting friction for remote developer teams. This guide goes deeper with platform-specific shortcuts, automation scripts for meeting prep, and etiquette guidelines for screen sharing, camera use, and post-meeting follow-up.

## Pre-Meeting Preparation

The foundation of meeting etiquette begins before the meeting starts. Taking a few minutes to prepare significantly improves meeting quality for everyone involved.

### Test Your Setup Before Joining

Technical failures during meetings waste everyone's time. Create a quick verification script to check your audio and video before every important call:

```bash
#!/bin/bash
# Quick meeting readiness check
echo "=== Meeting Setup Check ==="

# Test microphone
echo "Testing microphone..."
arecord -d 1 -q /dev/null && echo "✓ Microphone working" || echo "✗ Microphone issues"

# Test speakers
echo "Testing speakers..."
speaker-test -t sine -f 440 -l 1 -q 2>/dev/null && echo "✓ Speakers working" || echo "✗ Speaker issues"

# Check camera
echo "Checking camera..."
v4l2-ctl --list-devices 2>/dev/null | grep -q "Video" && echo "✓ Camera detected" || echo "✗ No camera found"

# Network latency check
ping -c 3 $(echo $(ip route | grep default | awk '{print $3}') | head -c -1) 2>/dev/null
echo "Network check complete"
```

On macOS, use the system audio settings to verify input and output devices. On Windows, the Sound settings provide a similar verification interface. Spending 30 seconds on this check prevents the awkward "can you hear me now?" exchanges that plague many meetings.

### Review the Agenda and Relevant Materials

When you're invited to a meeting with an agenda, actually read it. If no agenda exists, that itself is a red flag. For developer-focused meetings, this means:

- Review any PRs or code changes to be discussed
- Pull the latest changes if demo code is involved
- Test any tools or demos you'll be presenting
- Prepare specific questions or feedback points

This preparation shows respect for everyone's time and enables more productive discussions.

## During the Meeting: Core Etiquette Rules

### Camera Etiquette

Camera presence significantly impacts meeting dynamics. The general rule: keep your camera on for meetings with fewer than eight participants. For larger meetings, camera is optional but encouraged when speaking.

When your camera is on:

- Position your face so lighting comes from in front (not behind you)
- Center your face in the frame with some headroom
- Avoid distracting backgrounds—clean rooms or simple blur effects work best
- Look at the camera when speaking, not at the screen

```bash
# Simple OBS virtual camera setup for Linux
# This creates a filtered virtual camera for consistent appearance
obs --startrecording --scene "Presentation" --filter "color correction: brightness=1.1"
```

### Audio Best Practices

Mute when not speaking. This seems obvious, but many meetings suffer from background noise, keyboard typing, or echo from unmuted participants. Most platforms show clear mute indicators—use them:

- Zoom: ⌘+D (Mac) or Alt+M (Windows)
- Google Meet: ⌘+D (Mac) or Ctrl+D (Windows)
- Microsoft Teams: ⌘+Shift+M (Mac) or Ctrl+Shift+M (Windows)

When speaking, slightly lean toward your microphone for clarity. If using a headset with a boom microphone, position it two finger-widths from your mouth.

### Screen Sharing with Purpose

When screen sharing, close unnecessary applications and disable notifications. For developer meetings, this includes:

```bash
# macOS: Disable notifications before screen share
# Run this in Terminal
osascript -e 'tell application "System Events" to set do not disturb to true'
```

On Linux, use `notify-send --urgency=low "" ""` or disable notifications via your desktop environment's settings.

When presenting code:

- Use a syntax-highlighted editor (VS Code, Vim with plugins)
- Increase font size to at least 18pt
- Use a dark theme if the room is dimmed
- Use built-in zoom features rather than scaling the entire screen

### Meeting Participation for Developers

Active participation improves meeting outcomes. Use these strategies:

- Use the raised hand feature in larger meetings to signal you want to speak
- Use chat for code links instead of reading URLs aloud
- Confirm action items before the meeting ends—repeat back what you're responsible for
- Take visible notes—screen sharing your note-taking shows engagement

## Automation for Meeting Efficiency

Developers can automate many meeting-related tasks to improve consistency and save time.

### Calendar Integration

Set up automated meeting reminders that include prep tasks:

```python
# Example: Calendar event prep checklist generator
def generate_prep_checklist(meeting_title, has_code_review=False, has_demo=False):
    checklist = [
        f"Review agenda for: {meeting_title}",
        "Check audio/video setup",
        "Close unnecessary applications",
    ]

    if has_code_review:
        checklist.extend([
            "Pull latest changes",
            "Run tests locally",
            "Note specific feedback points"
        ])

    if has_demo:
        checklist.extend([
            "Test demo environment",
            "Prepare backup slides",
            "Verify screen share permissions"
        ])

    return checklist
```

### Automated Status Updates

For recurring meetings, use status scripts that prepare your environment:

```bash
#!/bin/bash
# Meeting mode: optimize system for video call
# Add to your path and run before meetings

# Close resource-heavy applications
pkill -f "chrome" 2>/dev/null || true
pkill -f "slack" 2>/dev/null || true

# Set do not disturb
defaults write com.apple.notificationcenterui doNotDisturb -boolean true

# Optimize audio
sudo sysctl -w net.inet.tcp.delayed_ack=0 2>/dev/null || true

echo "Meeting mode activated"
```

## Platform-Specific Tips

### Zoom

Zoom remains the most feature-rich option for developer meetings. Key shortcuts to memorize:

- Start/stop video: ⌘+V
- Mute/unmute: ⌘+Shift+M
- Share screen: ⌘+Shift+S
- New breakroom: ⌘+Option+B

Enable "HD" and "Touch up my appearance" in settings for better video quality.

### Google Meet

Google Meet integrates tightly with Google Workspace. For developers:

- Use the companion mode for secondary screen sharing
- Pin specific participants when reviewing code together
- Use the hand raise function for clearer turn-taking

### Microsoft Teams

Teams excels at meetings within the Microsoft ecosystem:

- Use the "Share tray" for specific window sharing (not full screen)
- Use the whiteboard feature for architectural discussions
- Use Together mode for better meeting engagement

### Jitsi and Self-Hosted Options

For privacy-conscious teams, Jitsi provides a capable free alternative:

```yaml
# docker-compose.yml for self-hosted Jitsi
services:
  jitsi:
    image: jitsi/web
    ports:
      - "80:80"
      - "443:443"
    environment:
      - ENABLE_AUTH=1
      - ENABLE_LOBBY=1
```

Self-hosting gives you control over recording, encryption, and data retention policies.

## Post-Meeting Etiquette

Meeting etiquette extends beyond the call itself:

- Send meeting notes within 24 hours if you're responsible
- Update action items in your project management tool immediately
- Share relevant recordings with timestamps for absent team members
- Clean up shared resources—close shared documents, end shared cursor sessions


## Frequently Asked Questions


**Are free AI tools good enough for practices: a developer guide?**

Free tiers work for basic tasks and evaluation, but paid plans typically offer higher rate limits, better models, and features needed for professional work. Start with free options to find what works for your workflow, then upgrade when you hit limitations.


**How do I evaluate which tool fits my workflow?**

Run a practical test: take a real task from your daily work and try it with 2-3 tools. Compare output quality, speed, and how naturally each tool fits your process. A week-long trial with actual work gives better signal than feature comparison charts.


**Do these tools work offline?**

Most AI-powered tools require an internet connection since they run models on remote servers. A few offer local model options with reduced capability. If offline access matters to you, check each tool's documentation for local or self-hosted options.


**Can I use these tools with a distributed team across time zones?**

Most modern tools support asynchronous workflows that work well across time zones. Look for features like async messaging, recorded updates, and timezone-aware scheduling. The best choice depends on your team's specific communication patterns and size.


**Should I switch tools if something better comes out?**

Switching costs are real: learning curves, workflow disruption, and data migration all take time. Only switch if the new tool solves a specific pain point you experience regularly. Marginal improvements rarely justify the transition overhead.


## Related Articles

- [Best Hybrid Meeting Etiquette Guide Ensuring Remote](/remote-work-tools/best-hybrid-meeting-etiquette-guide-ensuring-remote-particip/)
- [How to Create Remote Team Inclusive Meeting Practices Guide](/remote-work-tools/how-to-create-remote-team-inclusive-meeting-practices-guide-/)
- [Zoom CLI example for updating PMI settings](/remote-work-tools/best-virtual-meeting-room-for-recurring-remote-client-check-/)
- [Best Cafe Work Etiquette for Remote Workers](/remote-work-tools/best-cafe-work-etiquette-for-remote-workers/)
- [Best Practices for Async Pull Request Reviews on](/remote-work-tools/best-practices-for-async-pull-request-reviews-on-distributed/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
