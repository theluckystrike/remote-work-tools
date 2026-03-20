---
layout: default
title: "How to Reduce Fan Noise from Desktop PC During Video Calls"
description: "Practical techniques to minimize desktop PC fan noise during video calls. Includes software tweaks, fan curve configurations, and hardware."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-reduce-fan-noise-from-desktop-pc-during-video-calls/
categories: [guides]
tags: [hardware, noise-reduction, video-calls, remote-work]
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


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
