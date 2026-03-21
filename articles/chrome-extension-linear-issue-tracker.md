---
layout: default
title: "Chrome Extension Linear Issue Tracker: Practical Guide"
description: "Discover Chrome extensions that integrate with Linear for issue tracking. Learn how to improve your workflow with browser-based Linear access, quick"
date: 2026-03-15
last_modified_at: 2026-03-15
author: "Remote Work Tools Guide"
permalink: /chrome-extension-linear-issue-tracker/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

Install the official Linear browser extension to create issues from any webpage without context switching and preview issues directly in GitHub PRs. Linear is popular among development teams for its clean interface and GitHub integration, but Chrome extensions add capabilities that improve workflow efficiency—create issues without switching tabs, view issue previews in PRs, and access notifications directly in Chrome. This guide covers practical Linear extensions and how to integrate them into your daily development routine.

## Why Use Chrome Extensions with Linear

Linear's web application works well, but browser extensions add capabilities that improve productivity for developers who spend significant time in Chrome. These extensions can:

- Create issues from any webpage without switching contexts
- Display issue previews when viewing GitHub PRs
- Provide keyboard shortcuts for common actions
- Show Linear notifications alongside browser alerts

## Practical Chrome Extensions for Linear

### 1. Linear - Issues & Projects

The official Linear browser extension provides core functionality directly in Chrome. After installing, you can create issues, view your inbox, and access recent projects without opening a new tab.

Installation: Search for "Linear" in the Chrome Web Store or visit linear.app/downloads. Sign in with your Linear account to activate the extension.

Key Features:
- Quick issue creation from the extension popup
- View assigned issues and notifications
- Search across all workspaces
- Keyboard shortcut: `Cmd+Shift+L` (Mac) or `Ctrl+Shift+L` (Windows)

When you click the extension icon, a popup appears showing your inbox count and recent issues. This works well for a quick status check between coding sessions.

### 2. GitHub Linear Issue Connector

This extension bridges GitHub pull requests with Linear issues. When viewing a PR that references a Linear issue (like `LINE-123`), the extension displays the issue status directly in the GitHub UI.

Use Case: You're reviewing a PR and want to check if the linked issue is already resolved. Instead of opening Linear in a new tab, you see the issue status inline.

```javascript
// The extension detects patterns like LINEAR-123 in PR descriptions
// and fetches issue status from the Linear API
const issuePattern = /LINEAR-\d+/g;
const matches = prDescription.match(issuePattern);
// Displays issue status badge next to PR title
```

This is particularly useful for code reviewers who want to verify issue completion without context switching.

### 3. Linear Quick Add

Quick Add extensions let you create issues from anywhere in Chrome using a keyboard shortcut. This works when you're viewing documentation, a bug report, or any page that contains actionable information.

Workflow:
1. Navigate to a page with relevant information
2. Press `Cmd+Shift+Y` to open the quick-add dialog
3. The extension pre-fills the page URL and selected text as the issue description
4. Add a title, select project and labels, then create

This eliminates copy-pasting between tabs. The URL serves as context, making issues more actionable for whoever receives them.

### 4. Custom Extension: Issue Linker

For teams with specific workflows, building a custom Chrome extension that communicates with Linear's API offers maximum flexibility. Here's a basic implementation:

```javascript
// manifest.json
{
  "manifest_version": 3,
  "name": "Linear Issue Linker",
  "version": "1.0",
  "permissions": ["activeTab", "storage"],
  "host_permissions": ["https://linear.app/*"],
  "background": {
    "service_worker": "background.js"
  }
}

// background.js - Quick issue creation
chrome.action.onClicked.addListener(async (tab) => {
  const issueData = {
    title: "Issue from: " + tab.title,
    description: "Source: " + tab.url,
    teamId: "YOUR_TEAM_ID"
  };

  // Make API call to Linear
  const response = await fetch("https://api.linear.app/graphql", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "YOUR_API_KEY"
    },
    body: JSON.stringify({
      query: `
        mutation CreateIssue($input: IssueCreateInput!) {
          issueCreate(input: $input) {
            success
            issue {
              identifier
              url
            }
          }
        }
      `,
      variables: { input: issueData }
    })
  });
});
```

This example creates an issue from the current page's title and URL. Extend it to capture selected text, add labels, or assign to team members based on page content.

## Setting Up Your Extension Workflow

### Authentication

Most extensions require a Linear API key or OAuth connection. To generate an API key:

1. Open Linear and go to Settings → API
2. Click "Create new key"
3. Set appropriate permissions (read/write based on needs)
4. Store the key securely—never commit it to repositories

### Keyboard Shortcuts

Custom keyboard shortcuts make extensions feel native. Check the extension settings page (`chrome://extensions/shortcuts`) to configure:

| Action | Recommended Shortcut |
|--------|---------------------|
| Quick add issue | Cmd+Shift+I |
| Open Linear | Cmd+Shift+L |
| Search issues | Cmd+Shift+F |

### Integration with Development Workflow

Combine extensions with your existing tools for maximum efficiency:

Code Review: Use the GitHub connector to see Linear issue status while reviewing PRs. If the issue is marked "In Progress," request changes before merging.

Bug Reporting: When users report bugs in your app, use Quick Add to create issues immediately while the context is fresh. Include the URL and any console errors.

Documentation: Create issues for outdated documentation directly from docs.linear.app or your own wikis.

## Limitations and Alternatives

Chrome extensions work within browser constraints. For deeper integration, consider:

- Linear Desktop App: Native performance with system notifications
- VS Code Extension: Create issues without leaving your editor
- Slack Integration: Create issues from Slack messages

Extensions work best for quick actions and context-aware issue creation. Reserve complex issue management for the full Linear interface.

## Building Your Custom Extension: Complete Example

For teams wanting tighter Linear integration, building a simple custom extension beats any off-the-shelf solution:

```json
{
  "manifest_version": 3,
  "name": "Linear Issue Creator",
  "version": "1.0.0",
  "description": "Create Linear issues from anywhere in Chrome",
  "permissions": [
    "activeTab",
    "scripting",
    "storage",
    "webRequest"
  ],
  "host_permissions": [
    "https://linear.app/*",
    "https://api.linear.app/*",
    "https://*/*"
  ],
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "images/linear-16.png",
      "48": "images/linear-48.png",
      "128": "images/linear-128.png"
    }
  },
  "background": {
    "service_worker": "background.js"
  },
  "commands": {
    "open-creator": {
      "suggested_key": {
        "default": "Ctrl+Shift+I",
        "mac": "Cmd+Shift+I"
      },
      "description": "Create a new Linear issue"
    }
  }
}
```

Create `popup.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: system-ui; width: 400px; padding: 16px; }
    input, textarea, select { width: 100%; padding: 8px; margin: 8px 0; }
    button { background: #0066ff; color: white; padding: 10px; width: 100%; }
  </style>
</head>
<body>
  <h2>Create Issue</h2>
  <input id="title" type="text" placeholder="Issue title">
  <textarea id="description" placeholder="Description"></textarea>
  <select id="team">
    <option value="">Select team...</option>
  </select>
  <select id="priority">
    <option value="0">No priority</option>
    <option value="1">Urgent</option>
    <option value="2">High</option>
    <option value="3">Medium</option>
    <option value="4">Low</option>
  </select>
  <button id="create">Create Issue</button>
  <div id="status"></div>
  <script src="popup.js"></script>
</body>
</html>
```

Create `popup.js`:

```javascript
const API_KEY = 'YOUR_LINEAR_API_KEY'; // Store in Chrome storage in production

document.getElementById('create').addEventListener('click', async () => {
  const title = document.getElementById('title').value;
  const description = document.getElementById('description').value;
  const teamId = document.getElementById('team').value;
  const priority = parseInt(document.getElementById('priority').value);

  if (!title || !teamId) {
    document.getElementById('status').textContent = 'Please fill in all fields';
    return;
  }

  const query = `
    mutation CreateIssue($input: IssueCreateInput!) {
      issueCreate(input: $input) {
        issue {
          id
          identifier
          url
          title
        }
        success
      }
    }
  `;

  try {
    const response = await fetch('https://api.linear.app/graphql', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        query,
        variables: {
          input: {
            title,
            description,
            teamId,
            priority
          }
        }
      })
    });

    const data = await response.json();
    if (data.data?.issueCreate?.success) {
      const issue = data.data.issueCreate.issue;
      document.getElementById('status').innerHTML =
        `✓ Created <a href="${issue.url}" target="_blank">${issue.identifier}</a>`;
      // Clear form
      document.getElementById('title').value = '';
      document.getElementById('description').value = '';
    } else {
      document.getElementById('status').textContent = 'Error creating issue';
    }
  } catch (error) {
    document.getElementById('status').textContent = `Error: ${error.message}`;
  }
});

// Load teams on popup open
chrome.storage.local.get(['teams'], (result) => {
  if (result.teams) {
    const select = document.getElementById('team');
    result.teams.forEach(team => {
      const option = document.createElement('option');
      option.value = team.id;
      option.textContent = team.name;
      select.appendChild(option);
    });
  }
});
```

Store your API key securely:

```bash
# Never commit the actual key. Instead, create an options page where users provide their key
# Or use environment variables during build:
export LINEAR_API_KEY="your_key_here"
npm run build
```

## Linear Extension Workflow Optimization

Once you have extensions installed, optimize your daily workflow:

**Suggested keyboard shortcut mapping:**
- Cmd+Shift+I — Create issue from current page
- Cmd+Shift+L — Open Linear app in new tab
- Cmd+Shift+F — Search Linear issues

**Browser bookmark bar organization:**
```
Linear |
├── My Issues
├── Backlog
├── Search
└── Team
```

Each bookmark links to a filtered Linear view (e.g., `https://linear.app/team/issues?filter=assignee:me`).

**Workflow templates for common activities:**

When researching a bug, create an issue immediately with:
- Current URL in description (provides context)
- Reproduction steps if known
- Error message or logs
- Label: "bug" + "needs-investigation"

When reviewing documentation, create issues for:
- Outdated examples
- Missing sections
- Confusing explanations
- Link to the documentation page

When in code review, before commenting on a PR:
- Is this a blocking issue or a suggestion?
- If blocking, create a Linear issue and reference in PR comment
- If suggestion, comment directly

## Performance Tips for Extension Users

Extensions can slow browser startup if not optimized:

- Disable extensions you rarely use
- Grant Linear extension only permission on linear.app domains
- Check extension memory usage in chrome://extensions/
- If using custom extension, minimize background script activity

For teams of developers sharing a custom extension, publish to your internal Chrome Web Store:

```bash
# Create a .crx file for distribution
# Upload to your internal store at chrome.google.com/webstore (requires developer account)
# Or use policies to force install via chrome policies JSON
```

## Comparison: Extensions vs. Native Apps

| Feature | Extension | Linear App | VS Code Extension |
|---------|-----------|------------|------------------|
| Create issue | ✓ Fast | ✓ Full featured | ✓ Context aware |
| View details | △ Limited | ✓ Full | ✓ Full |
| Keyboard shortcuts | ✓ Global | ✓ App only | ✓ Global |
| Notification integration | △ Browser | ✓ System | ✓ IDE integrated |
| Offline access | ✗ | △ Limited | △ Limited |
| Setup complexity | Medium | None | Low |

Most efficient teams use Chrome extension + VS Code extension together.


## Related Articles

- [Chrome Security Headers Extension: A Practical Guide for](/remote-work-tools/chrome-security-headers-extension/)
- [Shortcut vs Linear: Issue Tracking Comparison for](/remote-work-tools/shortcut-vs-linear-issue-tracking-comparison/)
- [Chrome Extension Compress Images Before Upload: A](/remote-work-tools/chrome-extension-compress-images-before-upload/)
- [Chrome Extension Currency Converter for Shopping: A](/remote-work-tools/chrome-extension-currency-converter-shopping/)
- [Chrome Extension MLA Citation Generator: A Developer Guide](/remote-work-tools/chrome-extension-mla-citation-generator/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
