---
layout: default
title: "How to Set Up Hybrid Office Digital Signage Showing Room"
description: "A technical guide for developers building digital signage systems that display meeting room availability and calendar events in hybrid offices"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-hybrid-office-digital-signage-showing-room-availability-and-events/
categories: [guides]
intent-checked: true
voice-checked: true
reviewed: true
score: 8
tags: [remote-work-tools]---
---
layout: default
title: "How to Set Up Hybrid Office Digital Signage Showing Room"
description: "A technical guide for developers building digital signage systems that display meeting room availability and calendar events in hybrid offices"
date: 2026-03-16
last_modified_at: 2026-03-22
author: theluckystrike
permalink: /how-to-set-up-hybrid-office-digital-signage-showing-room-availability-and-events/
categories: [guides]
intent-checked: true
voice-checked: true
reviewed: true
score: 8
tags: [remote-work-tools]---

{% raw %}

Digital signage displaying real-time room availability and upcoming events solves a common pain point in hybrid offices: employees walking around looking for available meeting spaces. This guide walks through building a room availability display system using calendar APIs, a content backend, and display hardware. You'll get practical code patterns you can adapt to Google Calendar, Microsoft Graph, or any modern calendar system.

## Understanding the Core Requirements

Before writing code, identify what your signage needs to show:

1. Current room status: Available, occupied, or reserved soon
2. Next meeting details: Who booked the room and for how long
3. Daily event highlights: Company all-hands, team standups, or important deadlines
4. Visual indicators: Color-coded status (green for available, red for in-use)

The challenge is pulling data from your calendar system, processing it into display-friendly content, and pushing it to screens at regular intervals. Most organizations use either Google Calendar or Microsoft 365, so this guide covers both.

## Building the Calendar Integration

### Google Calendar Approach

If your organization uses Google Workspace, the Calendar API provides straightforward access to room bookings. You'll need a service account with domain-wide delegation or a regular OAuth flow.

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build
from datetime import datetime, timedelta

def get_google_calendar_service():
    """Initialize Google Calendar API client."""
    scopes = ['https://www.googleapis.com/auth/calendar.readonly']
    credentials = service_account.Credentials.from_service_account_file(
        'service-account.json',
        scopes=scopes
    )
    return build('calendar', 'v3', credentials=credentials)

def fetch_room_status(calendar_service, room_email, hours_ahead=2):
    """Fetch upcoming events for a specific room."""
    now = datetime.utcnow()
    end_time = now + timedelta(hours=hours_ahead)

    events_result = calendar_service.events().list(
        calendarId=room_email,
        timeMin=now.isoformat() + 'Z',
        timeMax=end_time.isoformat() + 'Z',
        singleEvents=True,
        orderBy='startTime'
    ).execute()

    events = events_result.get('items', [])

    # Determine current status
    current_event = None
    for event in events:
        start = datetime.fromisoformat(event['start']['dateTime'].replace('Z', '+00:00'))
        end = datetime.fromisoformat(event['end']['dateTime'].replace('Z', '+00:00'))

        if start <= now <= end:
            current_event = event
            break

    return {
        'room_email': room_email,
        'current_status': 'occupied' if current_event else 'available',
        'current_event': current_event,
        'upcoming_events': events
    }
```

This function returns the room's current state and all upcoming bookings within your lookahead window. The caller decides how to display this information.

### Microsoft Graph Approach

For Microsoft 365 environments, the Graph API provides similar functionality:

```python
import requests

def fetch_rooms_from_graph(access_token, room_list_id):
    """Fetch all rooms in a room list via Microsoft Graph."""
    endpoint = f"https://graph.microsoft.com/v1.0/places/microsoft.graph.roomList/{room_list_id}/rooms"

    headers = {
        'Authorization': f'Bearer {access_token}',
        'Content-Type': 'application/json'
    }

    response = requests.get(endpoint, headers=headers)
    return response.json().get('value', [])

def get_room_free_busy(graph_token, room_id):
    """Get free/busy status for a specific room."""
    endpoint = "https://graph.microsoft.com/v1.0/me/calendar/getSchedule"

    headers = {
        'Authorization': f'Bearer {graph_token}',
        'Content-Type': 'application/json'
    }

    payload = {
        "schedules": [room_id],
        "startTime": {
            "dateTime": datetime.utcnow().isoformat(),
            "timeZone": "UTC"
        },
        "endTime": {
            "dateTime": (datetime.utcnow() + timedelta(hours=2)).isoformat(),
            "timeZone": "UTC"
        }
    }

    response = requests.post(endpoint, headers=headers, json=payload)
    return response.json()
```

## Creating the Display Content

Once you have the calendar data, transform it into display-friendly content. A simple approach uses HTML templates rendered server-side:

```python
from datetime import datetime

def generate_room_display_html(room_data):
    """Generate HTML for a single room display."""
    status_color = '#22c55e' if room_data['current_status'] == 'available' else '#ef4444'
    status_text = 'AVAILABLE' if room_data['current_status'] == 'available' else 'IN USE'

    html = f"""
    <div class="room-display">
        <div class="status-banner" style="background-color: {status_color}">
            {status_text}
        </div>
        <h2>{room_data.get('room_name', 'Meeting Room')}</h2>
    """

    if room_data.get('current_event'):
        event = room_data['current_event']
        start = datetime.fromisoformat(event['start']['dateTime'].replace('Z', '+00:00'))
        end = datetime.fromisoformat(event['end']['dateTime'].replace('Z', '+00:00'))
        duration = (end - start).total_seconds() / 60

        html += f"""
        <div class="current-meeting">
            <h3>Now: {event.get('summary', 'Meeting')}</h3>
            <p>Ends in {int(duration)} minutes</p>
        </div>
        """

    if room_data.get('upcoming_events'):
        html += '<div class="upcoming"><h3>Coming Up:</h3><ul>'
        for event in room_data['upcoming_events'][:3]:
            start = datetime.fromisoformat(event['start']['dateTime'].replace('Z', '+00:00'))
            html += f"<li>{start.strftime('%H:%M')} - {event.get('summary', 'Busy')}</li>"
        html += '</ul></div>'

    html += '</div>'
    return html
```

This generates static HTML you can serve to any display endpoint. For dynamic updates without page refreshes, consider adding WebSocket connections or polling from the display client.

## Building the Event Aggregation Layer

Beyond individual room status, many offices want a dashboard showing company-wide events and highlights. Create an aggregation endpoint that pulls from multiple calendar sources:

```python
def aggregate_office_events(calendar_services, config):
    """Aggregate events from multiple calendars into a unified feed."""
    events = []

    # Pull from team calendars
    for calendar_id in config['team_calendars']:
        events.extend(fetch_calendar_events(calendar_services['primary'], calendar_id))

    # Pull from room calendars for booking patterns
    for room_email in config['room_emails']:
        room_status = fetch_room_status(calendar_services['primary'], room_email)
        if room_status['current_event']:
            events.append({
                'type': 'room_booking',
                'title': room_status['current_event'].get('summary'),
                'room': room_email,
                'source': 'calendar'
            })

    # Sort by start time and return
    events.sort(key=lambda x: x.get('start_time', ''))
    return events[:20]  # Return top 20 events
```

This gives you a single feed combining room bookings with team events—useful for lobby displays showing what's happening in the office today.

## Display Hardware and Client Options

For the display endpoint, you have several approaches:

1. Dedicated signage players: Hardware like BrightSign or Samsung Smart Signage Platform runs a browser-based client
2. Repurposed hardware: A Chromecast, Amazon Fire TV Stick, or old laptop running in kiosk mode
3. Native display APIs: Some platforms like Yodeck, Screenly, or Rise Vision provide APIs for pushing content

A simple Chromium-based client works for most scenarios:

```html
<!DOCTYPE html>
<html>
<head>
    <meta http-equiv="refresh" content="60">
    <style>
        body {
            font-family: system-ui, sans-serif;
            margin: 0;
            background: #111;
            color: #fff;
        }
        .room-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 20px;
            padding: 20px;
        }
        .room-card {
            background: #222;
            border-radius: 12px;
            padding: 20px;
            border-left: 8px solid #22c55e;
        }
        .room-card.occupied {
            border-left-color: #ef4444;
        }
    </style>
</head>
<body>
    <div id="content"></div>
    <script>
        async function updateDisplay() {
            const response = await fetch('/api/room-status');
            const rooms = await response.json();
            // Render room cards
        }
        updateDisplay();
        setInterval(updateDisplay, 60000);
    </script>
</body>
</html>
```

The meta refresh tag provides a simple fallback if JavaScript fails, while the interval ensures content updates every minute.

## Deployment Considerations

When deploying room availability signage, consider these operational factors:

Network topology: Place displays on a wired network when possible. WiFi congestion in busy offices causes content to stutter or fail loading. If using WiFi, ensure displays connect to the same VLAN as your API servers.

Update frequency: Fetch calendar data every 1-5 minutes. Calendar systems rate-limit API calls, so balance freshness against quota limits. Cache responses server-side and serve cached data to displays.

Fallback content: Always have a default view showing static information (building map, company values, or a clock) when the API is unreachable. Displays showing "loading" or blank screens look broken.

Timezone handling: Meeting rooms often display times in the local timezone, but your API server may run in UTC. Explicitly handle timezone conversion so meeting times match what users expect.

## Frequently Asked Questions

**How long does it take to set up hybrid office digital signage showing room?**

For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Complex configurations with custom requirements may take longer. Having your credentials and environment ready before starting saves significant time.

**What are the most common mistakes to avoid?**

The most frequent issues are skipping prerequisite steps, using outdated package versions, and not reading error messages carefully. Follow the steps in order, verify each one works before moving on, and check the official documentation if something behaves unexpectedly.

**Do I need prior experience to follow this guide?**

Basic familiarity with the relevant tools and command line is helpful but not strictly required. Each step is explained with context. If you get stuck, the official documentation for each tool covers fundamentals that may fill in knowledge gaps.

**Will this work with my existing CI/CD pipeline?**

The core concepts apply across most CI/CD platforms, though specific syntax and configuration differ. You may need to adapt file paths, environment variable names, and trigger conditions to match your pipeline tool. The underlying workflow logic stays the same.

**Where can I get help if I run into issues?**

Start with the official documentation for each tool mentioned. Stack Overflow and GitHub Issues are good next steps for specific error messages. Community forums and Discord servers for the relevant tools often have active members who can help with setup problems.

## Related Articles

- [Digital Signage for Hybrid Office Communication: A](/remote-work-tools/digital-signage-for-hybrid-office-communication/)
- [How to Set Up Conference Room Owl Camera for Hybrid](/remote-work-tools/how-to-set-up-conference-room-owl-camera-for-hybrid-meetings/)
- [Pin configuration](/remote-work-tools/how-to-design-mother-and-parent-room-for-hybrid-office-retur/)
- [Meeting Room Booking System for Hybrid Office 2026](/remote-work-tools/meeting-room-booking-system-for-hybrid-office-2026/)
- [Example ndss configuration snippet](/remote-work-tools/how-to-set-up-hybrid-office-guest-wifi-for-visitors-and-cont/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
