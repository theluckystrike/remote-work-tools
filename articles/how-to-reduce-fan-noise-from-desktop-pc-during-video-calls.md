---
layout: default
title: "How to Reduce Fan Noise from Desktop PC During Video Calls"
description: "Practical techniques to minimize desktop PC fan noise during video calls. Includes software tweaks, fan curve configurations, and hardware"
date: 2026-03-16
last_modified_at: 2026-03-16
author: theluckystrike
permalink: /how-to-reduce-fan-noise-from-desktop-pc-during-video-calls/
categories: [guides]
tags: [remote-work-tools, hardware, noise-reduction, video-calls, remote-work, best-of]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}
# How to Reduce Fan Noise from Desktop PC During Video Calls

Desktop PCs generate heat, and that heat requires active cooling. When you're on video calls, your machine often works harder than you realize—video encoding, background processes, and browser tabs all contribute to CPU and GPU load. The result: fans spin faster, and your colleagues hear that distracting whirring in the background.

This guide covers practical methods to reduce fan noise during video calls without sacrificing performance for your actual work. You'll find software tweaks, configuration examples, and hardware adjustments that work well for developers and power users.

## Why Your PC Gets Loud During Video Calls

Modern video conferencing applications like Zoom, Google Meet, and Microsoft Teams run continuously while you're in a call. They encode video, decode incoming streams, process audio, and maintain network connections—all simultaneously. On a desktop PC, this creates sustained CPU and GPU load that triggers your cooling system.

The culprits are predictable:

- Video encoding: Whether using hardware acceleration or software encoding, your CPU/GPU works to compress your camera feed
- Browser overhead: Running Chrome or Firefox with multiple tabs while on a call adds background processes
- Background applications: IDEs, terminal emulators, Docker containers, and CI/CD pipelines all generate heat
- Thermal throttling: When components get hot, they slow down—but your fans spin up to prevent that

Understanding these sources helps you target the right solutions.

## Software Solutions: Reduce Load and Control Fans

### Adjust Process Priority

One immediate fix involves lowering the priority of your video conferencing application. This doesn't stop it from working—it just tells your operating system to prioritize your actual work tasks first.

On Linux, you can use `nice` and `renice`:

```bash
# Start Zoom with lower priority
nice -n 10 zoom

# Or reduce priority of an already-running process
renice 10 -p $(pgrep -f "zoom")
```

On Windows, access Task Manager, right-click the video call process, and set Priority to "Below Normal" or "Low." This prevents the video app from competing with your compiler or development environment for CPU cycles.

### Configure Fan Curves in BIOS or Software

Most modern motherboards and graphics cards let you define fan curves—graphs that control fan speed based on temperature. By setting a more gradual curve, you can keep fans quieter during moderate loads.

Access your BIOS during boot (usually Delete or F2) and look for "Fan Control" or "Q-Fan." A typical quiet-friendly curve might look like:

| Temperature (°C) | Fan Speed (%) |
|------------------|---------------|
| 30 | 20 |
| 50 | 35 |
| 70 | 60 |
| 85 | 100 |

This keeps fans slow during light work and only ramps up when temperatures actually warrant it.

If your motherboard supports it, manufacturer software like ASUS AI Suite, MSI Afterburner, or Corsair iCUE provides more granular control without rebooting into BIOS.

### Use Hardware Video Encoding

Software video encoding (using your CPU) generates more heat than hardware encoding (using your GPU or dedicated encoder). Most video apps support hardware acceleration—enable it in your settings:

- Zoom: Settings → Video → Enable hardware acceleration
- Google Meet: Automatically uses hardware encoding when available
- Microsoft Teams: Settings → Devices → Make sure "Use hardware acceleration for video" is on

This simple change often reduces CPU load by 20-30% during calls.

### Limit Background Processes

Before joining a call, close unnecessary applications. A quick script can help on Linux:

```bash
#!/bin/bash
# Kill resource-heavy background processes before a call
pkill -f "chrome" || true
pkill -f "slack" || true
systemctl stop docker  # Stop Docker containers if not needed
```

Create a bash alias for quick execution:

```bash
alias join-call="~/scripts/call-prep.sh && zoom"
```

On Windows, use Process Lasso or simply close browser tabs and pause background downloads.

## Hardware Modifications: Quiet the Machine

### Upgrade Case Airflow

If your case has poor airflow, components run hotter and fans spin faster. Consider:

- Adding intake fans at the front (pulling cool air in)
- Ensuring exhaust fans at the rear/top work properly
- Removing cable clutter from the front of the case
- Checking that no vents are blocked

A well-ventilated case keeps components cooler at lower fan speeds.

### Replace Stock CPU Cooler

Stock CPU coolers from Intel and AMD are functional but noisy. Aftermarket options from be quiet!, Noctua, or Cryorig offer better cooling at lower noise levels. The Noctua NH-D15 remains a popular choice for quiet operation—it moves significant air while running at low RPM.

### Apply Better Thermal Paste

Thermal paste connects your CPU/GPU to their coolers. Old or poorly applied paste creates heat transfer bottlenecks. Clean and reapply with quality thermal paste like Thermal Grizzly Kryonaut or Arctic MX-4. This can lower temperatures by 5-15°C, allowing fans to run slower.

### Upgrade to Quiet Case Fans

Stock case fans often prioritize cost over silence. Replacement fans from Noctua, be quiet!, or Corsair LPX series offer better bearings (often fluid dynamic) and optimized blade designs. Even a single quiet 140mm fan can replace two louder 120mm fans while moving more air.

Look for fans rated below 20 dBA for truly quiet operation.

## Audio Processing: Mask Residual Noise

Sometimes you can't eliminate all fan noise. In those cases, audio processing helps:

### Use Noise Suppression in Your Video App

Most video conferencing tools include noise suppression:

- Zoom: Settings → Audio → Suppress persistent background noise (set to "Auto" or "Low")
- Microsoft Teams: Settings → Devices → Noise suppression → "Auto"
- Google Meet: Automatically applies noise reduction

### Apply System-Level Noise Suppression

For stronger suppression, use system-level tools:

- Windows: Krisp (free tier works well) or NVIDIA RTX Voice (if you have a recent GPU)
- Linux: PulseAudio module with noise cancellation, or use `noise-suppression-for-voice` via PipeWire

These tools apply real-time audio processing to remove fan noise before it reaches your call.

## Quick Checklist Before Your Next Call

1. Close unnecessary browser tabs and applications
2. Enable hardware video encoding in your video app
3. Lower video call app priority (via Task Manager or nice)
4. Run a quick script to pause Docker or other background services
5. Check that case fans aren't obstructed
6. Enable noise suppression in your video app

## Building a Quieter Development Environment

For developers spending hours on calls, investing time into a quieter setup pays dividends. The steps above—software configuration, fan curve tuning, and selective hardware upgrades—combine to create a system that stays quiet during meetings but still performs when you're compiling code or running tests.

Start with the free software tweaks. They take minutes and often provide immediate results. Then evaluate whether hardware upgrades make sense for your situation.

Remember: your setup doesn't need to be silent—your colleagues simply shouldn't hear your cooling system over your voice.

## Hardware Upgrade Cost-Benefit Analysis

Reducing fan noise often requires small hardware investments. Here's what each upgrade costs and what noise reduction it delivers:

| Upgrade | Cost | Noise Reduction | Effort | Best Value |
|---------|------|-----------------|--------|-----------|
| Thermal paste replacement | $10-20 | 5-15°C improvement (fans run slower) | Medium | High |
| Stock fan replacement (1x 140mm) | $20-40 | 3-5 dBA | Medium | High |
| CPU cooler upgrade (high-end) | $80-150 | 5-8 dBA | High | Medium |
| Case airflow improvement | $0-30 | 2-3 dBA | Low | High |
| Intake/exhaust fan additions | $40-60 | 3-5 dBA | Medium | High |
| GPU thermal pad replacement | $30 | 3-8 dBA reduction | High | Medium |
| Full system redesign/modding | $200+ | 8-12 dBA | Very high | Low |

Most developers see best results from thermal paste + one quiet fan replacement ($40-60 total, 10-15°C cooler systems). This typically eliminates call-disrupting noise without expensive CPU cooler replacement.

## Specific Quiet Fan Recommendations

Not all quiet fans are equal. Real-world options for developers:

**Best All-Around: Noctua NF-A14 PWM (140mm)**
- Cost: $25-35
- Noise: 13.8-19.8 dBA (very quiet)
- Airflow: 140.2 CFM (adequate for large case)
- Warranty: 6-year guarantee
- Install time: 5 minutes to replace existing fan

**Budget Option: Arctic P14 PWM (140mm)**
- Cost: $12-18
- Noise: 0.3 Sone (roughly 20 dBA)
- Airflow: 140 CFM
- Warranty: 6 years
- Install time: 5 minutes
- Trade-off: Slightly noisier than Noctua but 50% cheaper

**High-Performance: be quiet! Dark Rock Pro 4 (CPU cooler)**
- Cost: $80-110
- Noise: ~15 dBA at full load
- TDP: Handles up to 250W
- Warranty: 5 years
- Install time: 30-45 minutes
- Trade-off: Expensive but excellent for sustained loads (video calls + compiling)

**Laptop Alternative: External cooling pad (Havit HV-F2050)**
- Cost: $20-35
- Effectiveness: Reduces laptop temp 5-10°C, thereby reducing fan speed
- Noise: Pad itself is quiet; reduces laptop fan noise 2-3 dBA
- Trade-off: Only works for laptops; requires desk space

## Real Configuration Examples

### Minimal Setup (Zero Cost)

```bash
# Linux: Set conservative fan curve via BIOS
# Most modern systems support this without additional tools
# Access BIOS (typically Delete/F2 at boot), find "Q-Fan" or "Fan Control"
# Set curve: 30°C→20%, 40°C→30%, 50°C→40%, 70°C→80%, 85°C+→100%

# Windows: Use Task Manager to lower video app priority
# Open Task Manager → Find "zoom.exe" or "Teams.exe"
# Right-click → Details tab → Right-click process → Set Priority → Below Normal
```

**Result:** 2-3°C cooler, fans spin 5-10% slower. Takes 10 minutes. Often eliminates background noise.

### Mid-Range Setup ($60 investment)

```bash
# Step 1: Replace thermal paste
# Required: Thermal Grizzly Kryonaut ($8), isopropyl alcohol ($5), lint-free cloth
# Time: 30 minutes for CPU

# Step 2: Add one quiet intake fan (front of case)
# Cost: $20-30 for quality 140mm fan
# Time: 10 minutes

# Result: 8-12°C cooler, noticeable reduction in fan noise during calls
```

### Comprehensive Setup ($150 investment)

```bash
# Step 1: Thermal paste + cleanup
# Step 2: Replace all case fans with quiet 140mm fans (2-3 fans)
# Step 3: Improve cable management for better airflow
# Step 4: Verify BIOS fan curve is conservative

# Result: System runs 12-18°C cooler, almost silent during video calls
```

## Software Noise Suppression Tools: Detailed Comparison

When hardware changes aren't possible or sufficient:

| Tool | Platform | Cost | Effectiveness | Drawback |
|------|----------|------|----------------|----------|
| Krisp | Windows/Mac | Free tier (60 min/month) | 85-90% fan noise removal | Free tier limited; paid is $5/month |
| NVIDIA RTX Voice | Windows + NVIDIA GPU | Free | 90%+ noise suppression | Requires RTX card (newer GPUs only) |
| NoiseTorch | Linux | Free, open source | 85% fan noise removal | Requires PulseAudio; steeper learning curve |
| Voicemeeter | Windows | Free | 70-80% (with plugins) | Complex setup; multiple steps |
| OBS Noise Gate | Any platform + OBS | Free | 60-70% (cuts noise below threshold) | Creates "choppy" effect if not configured precisely |

For developers on Windows with RTX GPU: NVIDIA RTX Voice (free) is unbeatable. Install, enable, done.

For developers on Mac or without RTX: Krisp free tier ($0) covers 60 minutes monthly—sufficient for a few calls weekly.

For Linux: NoiseTorch (free, open source) beats everything else if you're comfortable with PulseAudio.

## Pre-Call Routine: 2-Minute Optimization

Experienced remote workers run this check before every important call:

```bash
# 1. Close unnecessary apps
killall chrome firefox slack spotify docker  # Or equivalent on your OS

# 2. Set process priorities (Windows via PowerShell)
Get-Process zoom | % { $_.PriorityClass = "BelowNormal" }

# 3. Pause background tasks
systemctl stop docker  # Stop containers
# Or pause Dropbox/OneDrive sync via UI

# 4. Enable hardware acceleration
# Zoom: Settings → Video → Hardware acceleration = ON
# Teams: Settings → Devices → Hardware acceleration = ON

# 5. Check case fans aren't blocked
# Quick visual inspection: no dust, no cables blocking intake

# 6. Enable system noise suppression
# Windows: Open Krisp, click microphone icon
# Mac: Same
# Linux: Enable NoiseTorch (pavucontrol)

# 7. Test audio before call
# Quick 10-second recording to verify noise isn't audible
```

Running this 2-minute routine prevents 95% of "hey, your fan is really loud" messages from colleagues.

## When to Invest vs When to Accept Noise

Consider your situation:

**Invest in hardware/optimization if:**
- You're on 5+ hours of calls daily
- Your role requires high-credibility calls (client presentations, interviews)
- Your team has given feedback about background noise
- You're in a long-term remote role

**Accept the noise if:**
- You're on calls 2-3 times per week
- Background noise is mild (colleagues can hear you fine)
- Hardware investments don't fit your budget
- You'll replace the PC in under a year anyway

The cost-benefit math: A $30 fan upgrade preventing even one "can you mute your fan?" message per month is worth it. A $150 CPU cooler upgrade is worth it only if you're in calls daily for years.


## Related Articles

- [Home Office Air Circulation Fan That Is Quiet for Calls](/remote-work-tools/home-office-air-circulation-fan-that-is-quiet-for-calls/)
- [Best Portable White Noise Speaker for Remote Parents Taking](/remote-work-tools/best-portable-white-noise-speaker-for-remote-parents-taking-calls-in-shared-spaces/)
- [Best Headset for Remote Work Video Calls: A Technical Guide](/remote-work-tools/best-headset-for-remote-work-video-calls/)
- [Best Keyboard for Quiet Typing During Video Calls in Open](/remote-work-tools/best-keyboard-for-quiet-typing-during-video-calls-open-offic/)
- [Best Lighting Setup for Video Calls in Basement Home Office](/remote-work-tools/best-lighting-setup-for-video-calls-in-basement-home-office/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
