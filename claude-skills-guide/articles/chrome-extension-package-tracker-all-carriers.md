---
layout: default
title: "How to Build a Chrome Extension Package Tracker for All Carriers"
description: "A comprehensive guide for developers and power users to create a Chrome extension that tracks packages across multiple shipping carriers."
date: 2026-03-15
author: theluckystrike
permalink: /chrome-extension-package-tracker-all-carriers/
---

Building a Chrome extension that tracks packages across all major shipping carriers requires understanding carrier APIs, manifest configuration, and cross-origin request handling. This guide walks through the technical implementation for developers looking to create a strong multi-carrier tracking solution.

## Understanding the Challenge

Package tracking seems simple on the surface—enter a tracking number and get status updates. However, each carrier (UPS, FedEx, USPS, DHL, Amazon) uses different tracking number formats, API endpoints, and response structures. A truly universal tracker must handle these differences while providing a consistent user experience.

The core challenge is normalizing data from multiple sources into a unified interface. Before writing any code, you need to understand the tracking number patterns and available APIs for each carrier you intend to support.

## Chrome Extension Architecture

A multi-carrier package tracker extension consists of three main components:

1. **Popup Interface** - User input for tracking numbers and display of results
2. **Background Service** - Handles API calls and data normalization
3. **Content Scripts** - Optional, for extracting tracking numbers from web pages

### Manifest Configuration

Your manifest.json defines the extension's capabilities:

```json
{
  "manifest_version": 3,
  "name": "Universal Package Tracker",
  "version": "1.0",
  "permissions": [
    "storage",
    "activeTab",
    "scripting"
  ],
  "host_permissions": [
    "https://*.ups.com/*",
    "https://*.fedex.com/*",
    "https://*.usps.com/*",
    "https://*.dhl.com/*"
  ],
  "action": {
    "default_popup": "popup.html",
    "default_icon": "icon.png"
  },
  "background": {
    "service_worker": "background.js"
  }
}
```

The host_permissions array is critical—each carrier domain must be explicitly declared to allow API requests from your extension.

## Carrier Detection and API Integration

The first step in your background script is detecting which carrier a tracking number belongs to. Each carrier uses specific patterns:

```javascript
const CARRIER_PATTERNS = {
  ups: /^1Z[A-Z0-9]{16}$/i,
  fedex: /^[0-9]{12,22}$/,
  usps: /^[0-9]{20,22}$|^(94|92|93)[0-9]{20,22}$/,
  dhl: /^[0-9]{10,11}$/
};

function detectCarrier(trackingNumber) {
  for (const [carrier, pattern] of Object.entries(CARRIER_PATTERNS)) {
    if (pattern.test(trackingNumber.replace(/\s/g, ''))) {
      return carrier;
    }
  }
  return null;
}
```

Once you've identified the carrier, construct the appropriate API request. Most carriers require authentication, which presents a challenge for client-side extensions.

### Handling API Authentication

Rather than embedding API keys in your extension (which exposes them to anyone who inspects your code), consider these approaches:

1. **Backend Proxy** - Your own server handles API authentication and forwards requests
2. **OAuth Flows** - For carriers supporting OAuth, implement the flow in your background script
3. **Public APIs** - Some carriers offer limited public tracking endpoints

For development and personal use, you can store API keys in extension storage with the understanding that determined users could extract them:

```javascript
async function fetchTrackingData(trackingNumber, carrier) {
  const carrierApiUrls = {
    ups: 'https://onlinetools.ups.com/track/v1/details',
    fedex: 'https://apis.fedex.com/track/v1/trackingnumbers',
    usps: 'https://api.usps.com/track/v1/{trackingNumber}'
  };

  const apiKey = await getApiKey(carrier);
  
  const response = await fetch(carrierApiUrls[carrier], {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiKey}`
    },
    body: JSON.stringify({
      includeDetailedScans: true,
      trackingInfo: [{ trackingNumberInfo: { trackingNumber } }]
    })
  });

  return response.json();
}
```

## Data Normalization

Each carrier returns tracking data in different formats. Create a normalization layer:

```javascript
function normalizeTrackingResponse(carrier, rawData) {
  const normalized = {
    carrier,
    status: 'unknown',
    events: [],
    estimatedDelivery: null,
    actualDelivery: null
  };

  switch (carrier) {
    case 'ups':
      normalized.status = rawData.trackResponse?.shipment?.[0]?.package?.[0]?.currentStatus?.description;
      normalized.events = (rawData.trackResponse?.shipment?.[0]?.package?.[0]?.activity || []).map(event => ({
        timestamp: `${event.date} ${event.time}`,
        location: event.location?.address?.city + ', ' + event.location?.address?.stateProvince,
        description: event.status?.description
      }));
      break;
      
    case 'fedex':
      normalized.status = rawData.output?.completeTrackResults?.[0]?.scanEvents?.[0]?.derivedStatus;
      normalized.events = rawData.output?.completeTrackResults?.[0]?.scanEvents?.map(event => ({
        timestamp: event.date,
        location: event.scanLocation?.city + ', ' + event.scanLocation?.stateOrProvinceCode,
        description: event.eventDescription
      }));
      break;
    // Additional carriers follow similar patterns
  }

  return normalized;
}
```

## Popup Interface Implementation

The popup provides the user-facing component. Use a clean, minimal design:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { width: 320px; font-family: system-ui, sans-serif; padding: 16px; }
    input { width: 100%; padding: 8px; margin-bottom: 12px; box-sizing: border-box; }
    button { width: 100%; padding: 10px; background: #0066cc; color: white; border: none; cursor: pointer; }
    .result { margin-top: 16px; padding: 12px; background: #f5f5f5; border-radius: 4px; }
    .event { padding: 8px 0; border-bottom: 1px solid #ddd; }
    .status { font-weight: bold; color: #0066cc; }
  </style>
</head>
<body>
  <h3>Package Tracker</h3>
  <input type="text" id="trackingInput" placeholder="Enter tracking number">
  <button id="trackBtn">Track Package</button>
  <div id="results"></div>
  <script src="popup.js"></script>
</body>
</html>
```

Connect the popup to your background script:

```javascript
document.getElementById('trackBtn').addEventListener('click', async () => {
  const trackingNumber = document.getElementById('trackingInput').value.trim();
  const resultsDiv = document.getElementById('results');
  
  resultsDiv.innerHTML = 'Loading...';
  
  // Send message to background script
  const response = await chrome.runtime.sendMessage({
    action: 'track',
    trackingNumber
  });
  
  if (response.error) {
    resultsDiv.innerHTML = `<p style="color: red">${response.error}</p>`;
    return;
  }
  
  renderResults(response.data, resultsDiv);
});
```

## Storage and Persistence

Allow users to save their tracked packages:

```javascript
async function savePackage(trackingData) {
  const { packages = [] } = await chrome.storage.local.get('packages');
  
  const exists = packages.find(p => p.trackingNumber === trackingData.trackingNumber);
  if (!exists) {
    packages.push({
      ...trackingData,
      addedAt: new Date().toISOString()
    });
    await chrome.storage.local.set({ packages });
  }
}
```

## Best Practices for Production

When moving beyond personal use, implement these considerations:

- **Rate Limiting** - Respect carrier API limits to avoid getting blocked
- **Error Handling** - Gracefully handle API failures, timeouts, and invalid tracking numbers
- **Caching** - Store recent results to reduce API calls and improve responsiveness
- **User Privacy** - Minimize data collection; consider local-only storage

Testing your extension requires loading it as an unpacked extension in Chrome's developer mode. Iterate on the UI based on real usage patterns.

## Conclusion

Building a Chrome extension for multi-carrier package tracking requires handling diverse APIs, normalizing data formats, and creating a smooth user experience. The architecture outlined here provides a foundation that scales—start with a few carriers and expand as you understand the data patterns.

The key to success is abstraction: separate carrier detection, API calls, and data normalization into distinct modules. This makes adding new carriers straightforward and keeps your code maintainable.

Built by theluckystrike — More at [zovo.one](https://zovo.one)
