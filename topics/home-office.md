---
layout: default
title: "Home Office Setup — Desks, Monitors, Ergonomics, and Equipment"
description: "Guides for building a productive home office: monitors, desks, chairs, lighting, audio, and ergonomics."
permalink: /topics/home-office/
---

# Home Office Setup

Guides and recommendations for building a productive home office: monitors, desks, standing desks, lighting, audio equipment, and ergonomics.

---

{% assign office = site.pages | where_exp: "p", "p.path contains 'articles/'" | where_exp: "p", "p.title != nil" | sort: "date" | reverse %}
{% for p in office limit: 30 %}{% if p.title contains 'Home Office' or p.title contains 'Monitor' or p.title contains 'Desk' or p.title contains 'Chair' or p.title contains 'Ergonomic' or p.title contains 'Lighting' or p.title contains 'Headset' or p.title contains 'Webcam' or p.title contains 'Keyboard' or p.title contains 'Standing' %}
- [{{ p.title }}]({{ p.url }})
{% endif %}{% endfor %}

[Back to home](/)
