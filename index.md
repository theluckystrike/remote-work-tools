---
layout: default
title: "Remote Work Tools"
description: "Reviews, comparisons, and guides for the best remote work tools, async workflows, and distributed team setups."
permalink: /
---

# Remote Work Tools

I have worked remotely for years and tried more tools than I can count. Most "best remote tools" lists are just affiliate link farms. This site is different -- every review comes from actual usage with real distributed teams. If a tool is good, I will tell you why. If it is not worth the price, I will tell you that too.

Whether you are setting up async workflows, choosing a project management tool, or building out a home office, start with the guides below.

## Start Here

The most useful guides for getting started:

- [Asana vs Linear for a 10-Person Dev Team](/asana-vs-linear-for-a-10-person-dev-team-comparison/)
- [Async Standup Alternative Using GitHub Commit Summaries](/async-standup-alternative-using-github-commit-summaries-automatically/)
- [Async Code Review Process Without Zoom Calls](/async-code-review-process-without-zoom-calls-step-by-step/)
- [Best 4K Monitor for Programming 2026](/best-4k-monitor-for-programming-2026/)
- [Best Accounting Software for Freelancers 2026](/best-accounting-software-for-freelancers-2026/)

---

## Topic Guides

Browse by focus area:

- [Video Conferencing Tools](/topics/video-conferencing-tools/) -- Zoom, hybrid meetings, screen sharing
- [Team Communication Tools](/topics/team-communication-tools/) -- Slack, Discord, Zulip, async messaging
- [Project Management Tools](/topics/project-management-tools/) -- Asana, Linear, Trello, ClickUp
- [Async Collaboration](/topics/async-collaboration-tools/) -- video messaging, RFC processes, standup alternatives
- [Home Office Setup](/topics/home-office-setup/) -- desks, monitors, ergonomics, equipment
- [Time Management Tools](/topics/time-management-tools/) -- tracking, productivity, time zones
- [Remote Security Tools](/topics/remote-security-tools/) -- VPN, zero trust, compliance
- [Remote Hiring and Onboarding](/topics/remote-hiring-onboarding/) -- ATS, interviews, onboarding checklists

---

## Recent Articles

{% assign all_articles = site.pages | where_exp: "p", "p.path contains 'articles/'" | sort: "title" %}
{% for p in all_articles limit:5 %}
- [{{ p.title }}]({{ p.url | relative_url }})
{% endfor %}

---

{{ all_articles.size }} articles and growing. [Browse all](/articles/) or read [about this site](/about/).
