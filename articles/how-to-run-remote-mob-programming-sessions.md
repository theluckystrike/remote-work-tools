---
layout: default
title: "How to Run Remote Mob Programming Sessions"
description: "Set up effective remote mob programming with VS Code Live Share, Tuple, and structured rotation — roles, timing, and anti-patterns to avoid"
date: 2026-03-22
author: theluckystrike
permalink: /how-to-run-remote-mob-programming-sessions/
categories: [guides]
tags: [remote-work-tools]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
---

{% raw %}

Mob programming — the whole team working on one thing together — sounds counterintuitive for remote teams, but it solves specific problems that async work can't: onboarding new engineers, tackling genuinely hard problems that need multiple perspectives simultaneously, and transferring knowledge across the team. Done well, a 90-minute remote mob session on a hard problem beats a week of back-and-forth async comments.

## The Core Structure

Remote mob programming needs more explicit structure than in-person because there's no shared physical space to anchor coordination.

**Roles:**
- **Driver**: types the code (shares screen, receives directions)
- **Navigator**: decides what to type next (the team collectively; one voice at a time)
- **Mob**: everyone else — observing, thinking ahead, flagging issues

**Timing:**
- 15-minute rotations (driver becomes part of mob, navigator becomes driver, mob member becomes navigator)
- Use a visible timer the whole group can see
- Short break every 90 minutes

**The golden rule**: The driver types only what the navigator says. No initiative, no "I think we should also..." — that goes through the navigator.

## Tool Setup

**Option 1: VS Code Live Share + Video Call**

Live Share lets multiple people edit the same file simultaneously, but in mob sessions only the driver types.

```bash
# Install Live Share extension
code --install-extension ms-vsliveshare.vsliveshare

# Session setup:
# 1. Navigator shares their VS Code session
# 2. Driver joins the share — they see the code and can type
# 3. Everyone else joins as read-only observers
# 4. Rotation: driver leaves, new driver joins as editor

# Dedicated VS Code workspace for mob sessions:
# .vscode/mob.code-workspace
{
  "settings": {
    "editor.fontSize": 16,
    "editor.lineHeight": 1.6,
    "editor.minimap.enabled": false,
    "liveshare.allowGuestDebugControl": true,
    "liveshare.allowGuestTaskControl": true
  }
}
```

**Live Share limitations:**
- Slow on large monorepos (file indexing latency)
- Terminal sharing requires explicit permission
- Extension version mismatches cause issues — everyone should be on the same VS Code version

**Option 2: Tuple**

Tuple is purpose-built for pair/mob programming with low-latency screen sharing and audio.

```bash
# Install Tuple (macOS only as of 2026)
brew install --cask tuple

# Start a mob session:
# 1. Host opens Tuple, invites all participants
# 2. Host shares screen
# 3. Rotation: host passes control to the driver with a click
# 4. Any participant can request control — the host approves

# Key Tuple features for mob sessions:
# - <100ms latency on good connections (vs 150-300ms in Zoom screen share)
# - Retina resolution screen sharing
# - Mouse multiplexing (multiple cursors visible)
# - Direct audio without video codec overhead
```

**Option 3: Gitpod or Codebase (Cloud IDE Mob)**

For teams across multiple operating systems or without powerful local machines:

```yaml
# .gitpod.yml — shared cloud dev environment
image:
  file: .gitpod.Dockerfile

tasks:
  - name: Setup
    init: |
      npm install
      cp .env.example .env
    command: npm run dev

ports:
  - port: 3000
    visibility: public

vscode:
  extensions:
    - ms-vsliveshare.vsliveshare
    - dbaeumer.vscode-eslint
    - esbenp.prettier-vscode
```

Everyone opens the same Gitpod workspace URL and uses Live Share within it — no local environment required.

## Session Structure (90 minutes)

```
:00 — Check-in (5 min)
  - What's the goal of this session? One sentence.
  - What's NOT in scope?
  - Who's first driver/navigator?
  - Timer visible to all?

:05 — First rotation (15 min)
  Navigator: "Let's start by reading the failing test"
  Driver: types and navigates as directed

:20 — Rotation (2 min)
  Timer bell → Driver pushes WIP commit → Navigator → Driver

:22 — Second rotation (15 min)

:37 — Third rotation (15 min)

:52 — Fourth rotation (15 min)

:67 — Retrospective (10 min)
  - What did we accomplish?
  - What should we do differently next session?
  - Any knowledge gaps surfaced?
  - Next session goal?

:77 — Final commit, push, and PR (10 min)
```

## WIP Commits for Rotation

Push a WIP commit on every rotation so the incoming driver starts with current code:

```bash
#!/bin/bash
# mob-handoff.sh — run at each rotation
git add -A
git commit -m "WIP: mob session $(date +%Y%m%d-%H%M) [$(git config user.name)]"
git push origin mob-session
echo "Handoff complete. Next driver: pull and open VS Code"
```

The mob session branch is squash-merged at the end with a proper commit message.

## Navigator Anti-Patterns

The most common failure mode in mob programming is navigator breakdown:

**"Let me just type it quickly"** — Navigator takes keyboard from driver. Stops the learning/discussion dynamic.

**"What do you think we should do?"** — Navigator asks the driver for decisions. The navigator decides; driver implements.

**Monologue navigator** — One person dominates navigation across rotations, turning it into pair programming with an audience. Explicitly require silent time for mob members to surface ideas.

**Too much detail** — "Type async def create_user open paren user: UserCreate close paren arrow UserResponse colon" — overly granular. Navigate at intention level: "Let's create the async endpoint for user creation with the standard signature."

## When Mob Programming Is Worth It

**High value:**
- Onboarding a new engineer to a complex area
- Debugging a production issue that's stumped multiple engineers
- Architectural decisions that need shared mental models
- Code review for a critical, complex PR
- Tackling a gnarly technical problem with no clear solution

**Low value / avoid:**
- Routine feature work that can be done independently
- Anything where one engineer is significantly more experienced (becomes tutoring)
- Work requiring deep focus with no collaboration benefits

## Tools for the Timer

```bash
# Terminal timer (visible to driver who shares screen)
# Install: brew install terminal-notifier
countdown() {
  local seconds=$1
  while [ $seconds -gt 0 ]; do
    echo -ne "\r\033[K⏱ $((seconds/60)):$(printf '%02d' $((seconds%60))) remaining"
    sleep 1
    ((seconds--))
  done
  terminal-notifier -title "Mob Session" -message "Rotation! Switch driver."
  echo -e "\r\033[K🔔 Time's up! Rotation."
}

# Start 15-minute timer
countdown 900
```

**Web-based alternatives:**
- [mobti.me](https://mobti.me) — free, shows timer to all participants if shared on screen
- [leantimer.com](https://www.leantimer.com) — mob-specific timer with role display

## Session Notes Template

```markdown
## Mob Session — [date]

**Goal:** [one sentence]
**Attendees:** @alice (nav1), @bob (driver1), @carol (driver2), @david (nav2)
**Duration:** 90 min

### What we built / decided:
- [specific outcome]

### Decisions made (link to ADR if significant):
- [decision and rationale]

### Knowledge transfers:
- @alice: explained the payment retry logic to @carol
- @david: unfamiliar with the test fixture setup — explained during session

### Follow-up items:
- [ ] #456 — refactor the session above into a clean PR (owner: @alice)
- [ ] Write tests for the edge case we discovered (owner: @bob)

### Next session goal (if applicable):
[what we'd tackle in a follow-up session]
```

## Related Reading

- [Async Pair Programming Workflow Using Recorded Walkthroughs](/async-pair-programming-workflow-using-recorded-walkthroughs-and-github/)
- [Async Code Review Process Without Zoom Calls](/async-code-review-process-without-zoom-calls-step-by-step/)
- [Remote Team Deployment Pipeline Best Practices](/remote-team-deployment-pipeline-best-practices/)

---

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
