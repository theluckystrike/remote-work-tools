---
layout: default
title: "How to Set Up Remote Pair Programming Workflow Guide"
description: "Configure VS Code Live Share, JetBrains Code With Me, and terminal-based pairing tools for effective remote pair programming sessions"
date: 2026-03-21
last_modified_at: 2026-03-21
author: "Remote Work Tools Guide"
permalink: /how-to-set-up-remote-pair-programming-workflow-guide/
categories: [guides]
tags: [remote-work-tools, programming, collaboration, workflow, remote-work]
reviewed: true
score: 9
voice-checked: true
intent-checked: true
---

{% raw %}

Use VS Code Live Share if your team already works in VS Code and needs instant pair programming with shared editing, cursor visibility, and integrated audio (free, included with VS Code). Use JetBrains Code With Me if you're in PyCharm, IntelliJ, or Goland and need IDE-native pairing with full language tooling (paid, $8.99/month or $89.99/year per participant). Use tmux + SSH if you need zero-dependency terminal-based pairing across Unix systems or need to pair on remote servers directly. This guide walks through setup, session management, and best practices for each tool.

## Why Pair Programming Works Better Remote

In-office pairing is natural: you share a screen and keyboard effortlessly. Remote pairing requires intentional tooling. The best tools give both participants equal control of the code, synchronized cursor visibility, and smooth audio for discussion. They also let the "driver" stay focused while the "observer" navigates and thinks strategically.

Effective remote pairing requires:
- **Low-latency cursor and selection sync** (ideally <100ms)
- **Both participants can edit simultaneously** (not just one)
- **Integrated voice or easy Slack integration** (not switching between apps)
- **Session history**: Saves you from losing unsaved work if connection drops
- **Lag-tolerant architecture**: Works on moderate bandwidth (>2Mbps sufficient)

## VS Code Live Share: Instant Pairing Without Setup

Live Share is built into VS Code and requires no additional download. One person starts a session and shares a link; the other opens the link in a browser or VS Code. Both see the same file, can edit simultaneously, and see each other's cursors.

### Installation and First Session

Live Share comes with VS Code 1.30+. Verify it's installed:

```
In VS Code, open Command Palette (Cmd+Shift+P / Ctrl+Shift+P)
Type "Live Share: Start Collaboration Session"
Click to start
```

VS Code generates a shareable link:
```
https://prod.liveshare.vscode.dev/join/ABC123DEF456
```

Share this link with your pairing partner. They can join from:
- VS Code (install Live Share extension if not already present)
- Web browser (no installation needed)
- Mobile browser (works, but limited functionality)

### Session Configuration

**Initial setup (once per user):**

1. Sign in with GitHub or Microsoft account (required for session management)
2. In VS Code settings, configure guest permissions:
   - Command Palette → Live Share: Change Session Options
   - Toggle "Allow Guests to Modify Files" (enabled by default)
   - Toggle "Allow Guests to See Your Cursor" (enabled)

**Per-session configuration:**

Once a session starts, the host can:
- Invite participants via the Live Share panel on the left sidebar
- Monitor which files guests are viewing
- End guest editing permissions without closing the session
- See guest cursor positions in real-time

### Audio and Chat

Live Share includes integrated audio calling:
```
In Live Share panel, click the phone icon to start audio
Participants auto-connect; no third-party tool needed
```

For text chat:
```
In Live Share panel, click chat icon
Messages appear inline; history visible during session
```

Most teams also use Slack huddles or Discord alongside Live Share. Keep audio on the pairing tool, text on Slack/Discord for asynchronous questions.

### Pair Programming Workflow

**Driver (person typing):**
```javascript
// Driver writes code while observer watches in real-time
function calculateTax(amount, rate) {
  // Observer sees cursor moving, can point out errors
  return amount * rate;
}
```

**Observer (thinking strategically):**
- Watches driver's cursor position and edits
- Spots logic errors before they become bugs
- Navigates the codebase in a separate editor window (same session)
- Asks clarifying questions via audio or chat

**Switch roles:**
```
Observer: "Let me take the keyboard. I'll add the error handling."
Driver clicks "Stop Editing" in Live Share panel
Observer's cursor becomes active; driver watches
```

### Terminal Sharing in Live Share

Live Share 1.24+ supports terminal sharing:

```
Host opens terminal in VS Code (Terminal → New Terminal)
Guests can see output in real-time
Only host can type in terminal (for security)
```

To let a guest type in the terminal:
1. Host opens Command Palette
2. Type "Live Share: Make Terminal Interactive"
3. Guest gains read-write access to terminal

**Use case:** Debugging a failing test. Observer runs the test, watches output, driver investigates.

### Performance and Limitations

**Strengths:**
- Zero setup for most developers (already have VS Code)
- Ultra-low latency (<100ms typical)
- Works over moderate bandwidth (2-5Mbps)
- Integrated audio is actually good quality
- Browser-based guest option is seamless

**Weaknesses:**
- Guests cannot control terminal by default (host-only by design)
- No cursor position history after session ends
- Guests cannot use all VS Code extensions (security restriction)
- Debugging support is limited (host can share debugging session, but guest cannot debug)

**Cost:** Free (included with VS Code, no paid tier)

**Bandwidth requirement:** 2-5Mbps is comfortable; 1Mbps is minimum

## JetBrains Code With Me: IDE-Native Pairing

If you're using PyCharm, IntelliJ IDEA, or other JetBrains IDEs, Code With Me is built in. It offers deeper IDE integration than Live Share: full debugging support, shared run configurations, and language-aware code completion across both participants.

### Installation and Session Start

Code With Me is bundled with JetBrains IDEs 2021.3+. In IntelliJ/PyCharm:

```
Menu → Tools → Code With Me → Start Session
JetBrains account login (required)
Session link generated
```

Share the link; participants click to join (opens in their IDE automatically if they have Code With Me installed, browser otherwise).

### Configuration: Host vs Guest Permissions

Before starting, set host-side permissions:

```
Tools → Code With Me → Session Settings
- Enable/disable guest editing (toggle "Allow Guests to Edit")
- Enable/disable guest terminal access
- Set session timeout (default 60 minutes)
```

Common workflow: Start with guests read-only, enable editing after discussing the plan.

### Shared Debugging

This is where Code With Me excels over Live Share:

**Host initiates debug session:**
```
Menu → Run → Debug (Shift+F9)
Breakpoints appear in guest editor in real-time
Guest sees variable values, stack trace, watches
Guest can hover variables to inspect values
```

**Guest can set breakpoints:**
- Click left margin of guest editor to add breakpoints
- Host's debugger respects guest-added breakpoints
- Both see execution step forward when either hits "Step Over"

**Example workflow:**

```python
# In PyCharm, host debugging; guest watching
def process_order(order_id):
    order = fetch_order(order_id)  # Breakpoint here
    # At this point, both host and guest see:
    # order = Order(id=123, total=99.99, items=[...])
    # Guest observes: "The items list is empty; should we fetch separately?"
    # Host steps forward, confirms the bug
```

### Terminal Sharing in Code With Me

Terminal sharing is more robust than Live Share:

```
Host opens Terminal in IDE
Guests can read output immediately
Toggle: Tools → Code With Me → Session Settings → "Allow terminal access"
Guests gain read-write access to terminal
```

Guests can run commands on the host's machine directly. Useful for:
- Running tests together
- Debugging CI/CD issues on host's system
- Executing build scripts

### Performance and Limitations

**Strengths:**
- Shared debugging (killer feature)
- IDE extensions work normally (not restricted like Live Share)
- Better handling of large files (Live Share can lag >500KB)
- Terminal access is straightforward
- VCS integration: both see git status, branches, changes

**Weaknesses:**
- Paid: $8.99/month or $89.99/year per user
- Requires JetBrains IDE (not for VS Code users)
- Latency slightly higher than Live Share (100-150ms typical)
- Guests must have a JetBrains account (free account okay)

**Cost:** $8.99/month or $89.99/year per user (includes all JetBrains IDEs on that license)

**Bandwidth requirement:** 3-8Mbps recommended

## Terminal Pairing: tmux + SSH for Zero Dependencies

For server-side development or when you need maximum control, tmux (terminal multiplexer) + SSH is the simplest approach. Both developers SSH into the same server and share a tmux session. No GUI, no latency, perfect for remote server work.

### Basic tmux Pair Setup

**Person A (host) creates a session:**
```bash
# On shared server
tmux new-session -s pair
# Session created: "pair"
# Person A is now in tmux shell
```

**Person B (guest) joins the session:**
```bash
# Person B SSHs to same server
ssh user@shared-server
# Then attaches to A's session
tmux attach-session -t pair
```

Now both see the exact same terminal. Both can type. Both see edits happen in real-time.

### Configuration for Better Pairing

**Increase history buffer:**
```bash
# In tmux config (~/.tmux.conf)
set -g history-limit 50000
```

**Enable mouse support (optional, easier for beginners):**
```bash
set -g mouse on
# Now you can click to position cursor, scroll with trackpad
```

**Create split panes for parallel work:**
```bash
# In tmux session, press Ctrl+B followed by:
% # Split pane vertically (side by side)
" # Split pane horizontally (top/bottom)

# Navigate between panes:
Ctrl+B arrow keys
```

**Example pairing workflow with splits:**
```
┌─────────────────────────┬──────────────────┐
│ Person A editing code   │ Person B in vim  │
│ (left pane)             │ (right pane)     │
│ test.py open            │ test.py open     │
│                         │                  │
│ Cursor at line 5        │ Cursor at line 12│
└─────────────────────────┴──────────────────┘
```

Both see the same file, but can navigate independently. Switch panes with Ctrl+B arrow keys.

### Advanced tmux Pairing Patterns

**Detach and re-attach without losing work:**
```bash
# Person A: Ctrl+B then D (detach)
# Terminal returns to shell, but session still active

# Person B continues working in the session

# Person A later:
tmux attach-session -t pair
# Re-enters the session at the same spot
```

**Multiple sessions for different features:**
```bash
# Session 1: Feature branch work
tmux new-session -s feature-auth

# Session 2: Bug fixes
tmux new-session -s bug-fix

# Person B can switch between sessions:
tmux attach-session -t feature-auth
# Then later:
tmux attach-session -t bug-fix
```

**Send commands to tmux session from outside:**
```bash
# Useful for CI/CD debugging
tmux send-keys -t pair "npm test" Enter
# This types "npm test" into the pair session and hits Enter
# Both people see the command execute
```

### Performance and Limitations

**Strengths:**
- Zero latency within local network
- No special software; tmux is standard on Unix systems
- Works on ancient internet speeds (<1Mbps)
- Full terminal control (run any command)
- Easier for server-side development

**Weaknesses:**
- Terminal-only (no GUI editors directly visible, but terminal vim/nano works)
- Requires SSH access to shared server (not possible for all teams)
- No integrated audio (use Slack/Discord alongside)
- Single cursor position (both see same cursor, harder to point at different areas)

**Cost:** Free (tmux is open source)

**Best for:** DevOps teams, server-side development, debugging production systems

## Comparison: When to Use Each Tool

| Scenario | Best Tool | Reason |
|----------|-----------|--------|
| Two VS Code users, remote | Live Share | Instant setup, integrated audio |
| Two PyCharm/IntelliJ users | Code With Me | Shared debugging, IDE integration |
| Multi-IDE team (VS Code + PyCharm) | Live Share | Works across IDEs via web |
| Debugging production server | tmux + SSH | Zero latency, full control |
| Pair review of large codebase | Code With Me | Better performance on large files |
| Async pairing (not real-time) | None; use comments in PR | Not live pairing; use code review instead |

## Session Management Checklist

**Before each session:**
- [ ] Test audio/video (if using Live Share audio)
- [ ] Agree on driver/observer roles
- [ ] Share session link; confirm guest can connect
- [ ] Disable notifications (Slack, email) to avoid interruptions
- [ ] Set IDE to fullscreen (Live Share, Code With Me)

**During session:**
- [ ] Switch roles every 15-20 minutes to avoid driver fatigue
- [ ] Use chat/audio for asking questions (not separate channels)
- [ ] Have guest set breakpoints if debugging
- [ ] Save frequently (especially in Live Share, which can time out)

**After session:**
- [ ] Push code changes to branch
- [ ] Create PR with pairing notes ("Pair with Jane on feature X")
- [ ] Close session gracefully (don't leave hanging connections)

## Conclusion

For VS Code teams: Use Live Share. For JetBrains teams: Use Code With Me. For server-side development: Use tmux + SSH. The best pairing tool is the one your team uses consistently; set a standard and stick with it. Remote pairing is as effective as in-office pairing when you have the right tool and practice clear communication patterns.

{% endraw %}
