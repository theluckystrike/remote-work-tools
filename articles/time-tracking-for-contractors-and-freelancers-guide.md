---
layout: default
title: "Time Tracking for Contractors and Freelancers Guide"
description: "A practical guide to time tracking for contractors and freelancers. Learn setup methods, automation techniques, and tools for accurate billing"
date: 2026-03-15
last_modified_at: 2026-03-22
author: "Remote Work Tools Guide"
permalink: /time-tracking-for-contractors-and-freelancers-guide/
categories: [guides]
tags: [remote-work-tools, time-tracking, freelancers, contractors, productivity]
reviewed: true
score: 8
intent-checked: true
voice-checked: true---
---
layout: default
title: "Time Tracking for Contractors and Freelancers Guide"
description: "A practical guide to time tracking for contractors and freelancers. Learn setup methods, automation techniques, and tools for accurate billing"
date: 2026-03-15
last_modified_at: 2026-03-22
author: "Remote Work Tools Guide"
permalink: /time-tracking-for-contractors-and-freelancers-guide/
categories: [guides]
tags: [remote-work-tools, time-tracking, freelancers, contractors, productivity]
reviewed: true
score: 8
intent-checked: true
voice-checked: true---

{% raw %}

The most effective time tracking methods for contractors and freelancers are plain-text log files with a parsing script for simplicity, ActivityWatch for automatic activity detection, Git Time Metric for development-integrated tracking, and CLI tools like timetrap for quick session timers. Combine multiple methods for maximum accuracy, and use a reconciliation script weekly to catch unbilled hours before invoicing.

## Key Takeaways

- **The best time tracking**: system is the one you actually use consistently.
- **Most tools support CSV**: or JSON export.
- **Combine multiple methods for**: maximum accuracy, and use a reconciliation script weekly to catch unbilled hours before invoicing.
- **For contractors and freelancers**: time equals revenue directly.
- **Tools like ActivityWatch (open**: source) run in the background and log window titles, active applications, and idle time.
- **Use automatic activity tracking**: as a background safety net, manual timers for billable client work, and Git-based tracking for development tasks.

## The Business Case for Precise Time Tracking

Every hour you fail to track is an hour you cannot bill. For contractors and freelancers, time equals revenue directly. Unlike salaried employees where hours blend together, your time has explicit monetary value. Clients expect detailed invoices with breakdown by task or project. Accurate time records protect both you and your client from disputes about billed hours.

Beyond invoicing, time tracking reveals patterns in your work. You discover which tasks consume more time than estimated, identify your peak productivity hours, and make data-driven decisions about pricing and project scoping.

## Manual Time Tracking Methods

For developers who prefer simplicity, plain text files work surprisingly well. Create a timestamped log using a simple format:

```bash
# At start of work session
echo "$(date '+%Y-%m-%d %H:%M') - START - Project Alpha" >> time.log

# At end of session
echo "$(date '+%Y-%m-%d %H:%M') - END - Project Alpha" >> time.log
```

This produces a simple log file you can parse later. A Python script can calculate total hours:

```python
import re
from datetime import datetime

def parse_time_log(filename):
    with open(filename) as f:
        lines = f.readlines()

    sessions = []
    for i in range(0, len(lines) - 1, 2):
        start = datetime.strptime(lines[i].split(' - ')[0], '%Y-%m-%d %H:%M')
        end = datetime.strptime(lines[i+1].split(' - ')[0], '%Y-%m-%d %H:%M')
        duration = (end - start).total_seconds() / 3600
        sessions.append(duration)

    return sum(sessions)

print(f"Total hours: {parse_time_log('time.log'):.2f}")
```

This approach requires discipline but gives you complete control over your data. No subscriptions, no cloud services, no vendor lock-in.

## Automated Time Tracking with Activity Detection

For developers who forget to start timers, automatic activity tracking fills the gap. Tools like ActivityWatch (open source) run in the background and log window titles, active applications, and idle time.

Install ActivityWatch on macOS via Homebrew:

```bash
brew install activitywatch
```

Configure it to track specific categories:

```python
# .config/activitywatch/aw-watcher-window.toml
[aw-watcher-window]
name = "aw-watcher-window"
poll_time = 5
exclude_title = ["ActivityWatch", "Spotify", "Discord"]
```

The watcher captures window titles which you can map to projects. Create a mapping file:

```json
{
  "project_mapping": {
    "VS Code": "client-alpha",
    "Terminal": "internal-tools",
    "Slack": "communication",
    "Figma": "design-work"
  }
}
```

This generates detailed reports showing exactly where your time goes. Export data weekly to generate client-ready invoices.

## Git-Based Time Tracking for Developers

Since developers already use Git constantly, tracking time through commit messages integrates naturally into existing workflows. Tools like Git Time Metric (GTM) attach time spent to each file automatically.

Install GTM:

```bash
# macOS
brew install git-time-metric

# Initialize in your repo
git time-metric init
```

Enable tracking:

```bash
git time-metric enable
```

The tool monitors file access and editing, then calculates time per file. Generate reports:

```bash
git time-metric report --format=json
```

Output shows time spent per file:

```json
{
  "files": {
    "src/main.rs": {
      "time": 3600,
      "commits": 12
    },
    "src/utils.rs": {
      "time": 1800,
      "commits": 5
    }
  },
  "total_time": 5400
}
```

Combine this with client or project-based repositories to separate billing categories cleanly.

## Command-Line Timers for Quick Sessions

When you need a quick timer without full automation, command-line tools provide instant feedback. The `timer` command (available via Homebrew) offers simple countdown and stopwatch functionality:

```bash
# Install
brew install timer

# Start a 25-minute pomodoro
timer -m 25

# Count up from zero
timer
```

For more sophisticated CLI tracking, consider `timetrap`:

```bash
gem install timetrap

# Start tracking
timetrap start "Client Project"

# Switch to different task
timetrap switch "Code Review"

# Display timesheet
timetrap display
```

This Ruby gem stores data in SQLite locally, giving you a searchable database of all tracked time.

## Combining Methods for Maximum Accuracy

The most reliable approach combines multiple tracking methods. Use automatic activity tracking as a background safety net, manual timers for billable client work, and Git-based tracking for development tasks. Periodically compare outputs to catch discrepancies.

Create a simple reconciliation script:

```python
import json
from datetime import datetime, timedelta

def reconcile_time_logs(automatic_log, timetrap_export, git_metrics):
    """Compare time from multiple sources and flag discrepancies."""
    discrepancies = []

    # Check if automatic tracking shows significantly more time
    auto_total = sum(s['hours'] for s in automatic_log)
    manual_total = sum(s['hours'] for s in timetrap_export)

    if auto_total > manual_total * 1.2:
        discrepancies.append({
            'type': 'untracked_time',
            'auto_hours': auto_total,
            'manual_hours': manual_total,
            'message': '20%+ untracked time detected'
        })

    return discrepancies

# Example usage
with open('auto.json') as f:
    auto_data = json.load(f)
with open('timetrap.json') as f:
    timetrap_data = json.load(f)
with open('git.json') as f:
    git_data = json.load(f)

issues = reconcile_time_logs(auto_data, timetrap_data, git_data)
for issue in issues:
    print(f"Warning: {issue['message']}")
```

Run this weekly to ensure your billing stays accurate.

## Data Export and Invoice Generation

Once tracking is in place, generate invoices from your data. Most tools support CSV or JSON export. Create a simple invoice generator:

```python
from datetime import datetime

def generate_invoice(time_entries, hourly_rate, client_name):
    total_hours = sum(entry['hours'] for entry in time_entries)
    total_amount = total_hours * hourly_rate

    invoice = f"""INVOICE
Client: {client_name}
Date: {datetime.now().strftime('%Y-%m-%d')}
Rate: ${hourly_rate}/hour

Time Entries:
"""
    for entry in time_entries:
        invoice += f"  {entry['date']}: {entry['hours']:.2f}h - {entry['description']}\n"

    invoice += f"""
Total Hours: {total_hours:.2f}
Total Due: ${total_amount:.2f}
"""
    return invoice

# Usage
entries = [
    {'date': '2026-03-10', 'hours': 4.5, 'description': 'API integration'},
    {'date': '2026-03-11', 'hours': 3.0, 'description': 'Bug fixes'},
]
print(generate_invoice(entries, 150, "Acme Corp"))
```

This outputs a clean invoice ready to send to clients.

## Choosing Your Tracking Approach

Start with the simplest method that fits your workflow. If you already use Git for development, add GTM and get time tracking with minimal behavior change. If you work across many applications, automatic tracking with ActivityWatch provides coverage. For pure simplicity, a CLI timer or even a text file works perfectly.

The best time tracking system is the one you actually use consistently. Experiment with different approaches until you find the rhythm that works for your specific situation.

## Frequently Asked Questions

**How long does it take to complete this setup?**

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

- [Best Time Tracking Tools for Remote Freelancers](/remote-work-tools/best-time-tracking-tools-for-remote-freelancers/)
- [Best Time Tracking Tool for a Solo Remote Contractor 2026](/remote-work-tools/best-time-tracking-tool-for-a-solo-remote-contractor-2026/)
- [How to Set Up Harvest for Remote Agency Client Time Tracking](/remote-work-tools/how-to-set-up-harvest-for-remote-agency-client-time-tracking/)
- [Get recent workflow run durations](/remote-work-tools/remote-engineering-team-build-time-tracking-as-developer-pro/)
- [Remote Team Support Ticket First Response Time Tracking for](/remote-work-tools/remote-team-support-ticket-first-response-time-tracking-for-/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
