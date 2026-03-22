---
layout: default
title: "Google Scholar Chrome Extension Development Guide"
description: "A practical guide to building and using Chrome extensions for Google Scholar. Covers Manifest V3, content scripts, and real-world implementation"
date: 2026-03-15
last_modified_at: 2026-03-22
author: "Remote Work Tools Guide"
permalink: /google-scholar-chrome-extension/
reviewed: true
score: 8
categories: [guides]
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

<<<<<<< HEAD
# Google Scholar Chrome Extension Development Guide

Google Scholar is the starting point for most academic and technical research, but its interface lacks features that serious researchers need daily: bulk citation export, integration with reference managers, PDF annotation, and Zotero/Mendeley sync. Chrome extensions fill these gaps. This guide covers both using existing Scholar extensions and building your own with Manifest V3.

## Recommended Google Scholar Extensions

Before building, check what already exists:

**Scholarcy** — AI-powered paper summaries. Generates a structured summary of any research paper with key findings, limitations, and methodology in seconds. Best for quickly triaging whether a paper deserves a full read.

**Zotero Connector** — The standard for reference management. One-click save of any Scholar result to your Zotero library, including metadata, PDF if available, and full citation. Essential for systematic literature reviews.

**Unpaywall** — Finds legally free PDF versions of papers. Works on Scholar results automatically — if a free version exists on an institutional server or preprint archive, Unpaywall links you to it.

**Open Access Button** — Similar to Unpaywall, but also lets you request a copy from the author directly when no open access version exists.

**Connected Papers** — Build visual citation graphs. Not a Scholar extension per se, but integrates via DOI and helps map the research landscape around a paper.

## Building a Custom Scholar Extension with Manifest V3

If existing extensions don't cover your workflow, here's how to build one. The most common use case: extracting structured data from Scholar results for custom processing or analysis.

### Project Setup

```bash
mkdir scholar-extension && cd scholar-extension
npm init -y
npm install -D webpack webpack-cli copy-webpack-plugin
```

### Manifest V3 Configuration

```json
// manifest.json
{
  "manifest_version": 3,
  "name": "Scholar Research Assistant",
  "version": "1.0.0",
  "description": "Enhanced Google Scholar research tools",

  "permissions": [
    "storage",
    "tabs",
    "activeTab"
  ],

  "host_permissions": [
    "https://scholar.google.com/*"
  ],

  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    }
  },

  "content_scripts": [{
    "matches": ["https://scholar.google.com/*"],
    "js": ["content.js"],
    "css": ["content.css"],
    "run_at": "document_end"
  }],

  "background": {
    "service_worker": "background.js"
  }
}
```

### Content Script: Extracting Paper Metadata

The content script runs on Scholar pages and extracts structured metadata from search results:

```javascript
// content.js — runs on scholar.google.com pages

/**
 * Extract paper data from a Scholar result element
 */
function extractPaperData(resultElement) {
  const titleEl = resultElement.querySelector('.gs_rt a, .gs_rt span')
  const authorsEl = resultElement.querySelector('.gs_a')
  const snippetEl = resultElement.querySelector('.gs_rs')
  const citedByEl = resultElement.querySelector('a[href*="cites="]')
  const yearEl = resultElement.querySelector('.gs_a')

  // Parse authors and year from the combined author string
  const authorText = authorsEl?.textContent || ''
  const yearMatch = authorText.match(/\b(19|20)\d{2}\b/)

  // Extract PDF link if present
  const pdfLinkEl = resultElement.querySelector('.gs_or_ggsm a')
  const scholarLink = resultElement.querySelector('.gs_rt a')

  return {
    title: titleEl?.textContent?.trim() || '',
    authors: authorText.split('-')[0]?.trim() || '',
    year: yearMatch ? parseInt(yearMatch[0]) : null,
    citedBy: citedByEl ? parseInt(citedByEl.textContent.replace(/\D/g, '')) : 0,
    snippet: snippetEl?.textContent?.trim() || '',
    pdfUrl: pdfLinkEl?.href || null,
    scholarUrl: scholarLink?.href || null,
  }
}

/**
 * Scrape all results on the current Scholar page
 */
function scrapeCurrentPage() {
  const results = document.querySelectorAll('.gs_r.gs_or.gs_scl')
  return Array.from(results).map(extractPaperData).filter(p => p.title)
}

// Listen for messages from popup
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.action === 'scrapePage') {
    const papers = scrapeCurrentPage()
    sendResponse({ papers })
  }

  if (message.action === 'exportCsv') {
    const papers = scrapeCurrentPage()
    const csv = papersToCSV(papers)
    downloadCSV(csv, 'scholar-results.csv')
    sendResponse({ success: true })
  }

  return true // Keep message channel open for async response
})

/**
 * Convert paper array to CSV
 */
function papersToCSV(papers) {
  const headers = ['Title', 'Authors', 'Year', 'Cited By', 'Scholar URL', 'PDF URL']
  const rows = papers.map(p => [
    `"${p.title.replace(/"/g, '""')}"`,
    `"${p.authors.replace(/"/g, '""')}"`,
    p.year || '',
    p.citedBy,
    p.scholarUrl || '',
    p.pdfUrl || ''
  ])

  return [headers, ...rows].map(row => row.join(',')).join('\n')
}

/**
 * Trigger CSV download in the browser
 */
function downloadCSV(content, filename) {
  const blob = new Blob([content], { type: 'text/csv' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  a.click()
  URL.revokeObjectURL(url)
}
```

### Popup UI
=======
## Chrome Extensions for Google Scholar: Why Build One

## Table of Contents

- [Chrome Extensions for Google Scholar: Why Build One](#chrome-extensions-for-google-scholar-why-build-one)
- [What Your Scholar Extension Should Do](#what-your-scholar-extension-should-do)
- [Architecture: Building Your Scholar Extension](#architecture-building-your-scholar-extension)
- [Publishing Your Extension](#publishing-your-extension)
- [Testing Before Publishing](#testing-before-publishing)
- [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)
- [Real Workflow: Using Your Scholar Extension](#real-workflow-using-your-scholar-extension)
- [Building Additional Features: Export to Zotero](#building-additional-features-export-to-zotero)
- [Team Exercise: Planning Your Extension (60 minutes)](#team-exercise-planning-your-extension-60-minutes)

Google Scholar is the default for academic/research lookups. Building a Chrome extension can enhance Scholar with features it lacks: highlight papers you've read, export citations in one click, show related papers, link to free PDF versions, track papers you've saved.

For remote researchers, librarians, and academics, a well-built Scholar extension saves hours per month.

## What Your Scholar Extension Should Do

**Core features**:
1. **One-click citation export** (BibTeX, APA, MLA)
2. **Mark papers read/interesting** (persist across sessions)
3. **Find free PDF links** (integrate with unpaywall.org)
4. **Show author reputation** (h-index, citations)
5. **Quick notes** (annotate results directly)

**Nice-to-have features**:
1. **Related papers widget** (show similar research)
2. **Institutional access check** (highlight papers your library has)
3. **Save to Notion/Zotero** (one-click sync)
4. **Citation count tracker** (graph citation growth over time)

## Architecture: Building Your Scholar Extension

### Manifest V3 Setup

Chrome moved to Manifest V3 in January 2024. Older V2 extensions no longer work. Every new Scholar extension must use V3.

```json
{
  "manifest_version": 3,
  "name": "Scholar Plus",
  "version": "1.0.0",
  "description": "Enhance Google Scholar with citation export and saved papers",
  "permissions": [
    "storage",
    "activeTab",
    "scripting",
    "host_permissions"
  ],
  "host_permissions": [
    "https://scholar.google.com/*"
  ],
  "action": {
    "default_popup": "popup.html",
    "default_title": "Scholar Plus"
  },
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [
    {
      "matches": ["https://scholar.google.com/*"],
      "js": ["content.js"],
      "css": ["content.css"]
    }
  ]
}
```

### Content Script: Enhance Scholar Pages

Content scripts run on Scholar pages and inject buttons/UI.

```javascript
// content.js
// Runs on Google Scholar results pages

function enhanceScholarResults() {
  // Find all result containers
  const results = document.querySelectorAll('div[data-cid]');

  results.forEach(result => {
    const paperId = result.getAttribute('data-cid');
    const titleElement = result.querySelector('h3 a');
    const title = titleElement?.textContent || 'Unknown';

    // Create enhancement UI
    const controls = document.createElement('div');
    controls.className = 'scholar-plus-controls';
    controls.innerHTML = `
      <button class="scholar-btn save-btn" data-id="${paperId}">
        ⭐ Save
      </button>
      <button class="scholar-btn cite-btn" data-id="${paperId}">
        📋 Cite
      </button>
      <button class="scholar-btn pdf-btn" data-id="${paperId}">
        📄 Find PDF
      </button>
    `;

    // Insert controls below title
    titleElement?.parentElement?.appendChild(controls);

    // Add event listeners
    controls.querySelector('.save-btn').addEventListener('click', () => {
      savePaper(paperId, title);
    });

    controls.querySelector('.cite-btn').addEventListener('click', () => {
      showCitationFormats(paperId, title);
    });

    controls.querySelector('.pdf-btn').addEventListener('click', () => {
      findFreeFullText(paperId, title);
    });
  });
}

// Save paper to local storage
function savePaper(paperId, title) {
  chrome.storage.local.get('savedPapers', (result) => {
    const saved = result.savedPapers || [];
    if (!saved.find(p => p.id === paperId)) {
      saved.push({
        id: paperId,
        title: title,
        savedAt: new Date().toISOString(),
        notes: ''
      });
      chrome.storage.local.set({ savedPapers: saved });
      showNotification('Paper saved!');
    }
  });
}

// Show citation formats popup
function showCitationFormats(paperId, title) {
  const formats = {
    bibtex: generateBibTeX(paperId, title),
    apa: generateAPA(title),
    mla: generateMLA(title)
  };

  chrome.runtime.sendMessage({
    action: 'showCitation',
    formats: formats
  });
}

// Find free PDF via unpaywall.org
async function findFreeFullText(paperId, title) {
  try {
    const doi = extractDOI(paperId);
    if (!doi) {
      showNotification('Could not extract DOI');
      return;
    }

    const response = await fetch(`https://api.unpaywall.org/v2/${doi}?email=user@example.com`);
    const data = await response.json();

    if (data.is_oa && data.oa_locations[0]?.url_for_pdf) {
      window.open(data.oa_locations[0].url_for_pdf, '_blank');
    } else {
      showNotification('No free version found');
    }
  } catch (error) {
    console.error('PDF search error:', error);
  }
}

// Run enhancement when page loads
if (document.readyState === 'loading') {
  document.addEventListener('DOMContentLoaded', enhanceScholarResults);
} else {
  enhanceScholarResults();
}

// Watch for new results (infinite scroll)
const observer = new MutationObserver(enhanceScholarResults);
observer.observe(document.body, {
  childList: true,
  subtree: true
});
```

### Citation Generation

Generate citations in common formats:

```javascript
// Generate BibTeX
function generateBibTeX(paperId, title) {
  const authors = extractAuthors(); // Parse from page
  const year = extractYear(); // Parse from page
  const key = `${authors[0]}_${year}`.replace(/[^a-z0-9]/gi, '');

  return `@article{${key},
    title={${title}},
    author={${authors.join(' and ')}},
    year={${year}}
  }`;
}

// Generate APA
function generateAPA(title) {
  const authors = extractAuthors();
  const year = extractYear();
  return `${authors.join(', ')} (${year}). ${title}.`;
}

// Generate MLA
function generateMLA(title) {
  const authors = extractAuthors();
  const year = extractYear();
  return `${authors.join(', ')}. "${title}." ${year}.`;
}
```

### Popup UI: Show Saved Papers
>>>>>>> 957a05ec9ec85ac69b64fcda12b5f2b7f2d068ca

```html
<!-- popup.html -->
<!DOCTYPE html>
<html>
<head>
<<<<<<< HEAD
  <meta charset="utf-8">
  <style>
    body { width: 300px; padding: 16px; font-family: system-ui; }
    button {
      display: block; width: 100%; padding: 8px;
      margin: 8px 0; cursor: pointer;
      background: #1a73e8; color: white; border: none;
      border-radius: 4px; font-size: 14px;
    }
    button:hover { background: #1557b0; }
    #status { font-size: 12px; color: #666; margin-top: 8px; }
  </style>
</head>
<body>
  <h3 style="margin-top: 0">Scholar Assistant</h3>
  <button id="export-csv">Export Results as CSV</button>
  <button id="export-bibtex">Export as BibTeX</button>
  <button id="copy-titles">Copy All Titles</button>
  <div id="status"></div>
=======
  <meta charset="UTF-8">
  <style>
    body {
      width: 400px;
      padding: 10px;
      font-family: -apple-system, BlinkMacSystemFont, sans-serif;
    }
    .tab-buttons {
      display: flex;
      gap: 10px;
      margin-bottom: 15px;
    }
    button {
      padding: 8px 12px;
      border: none;
      background: #f0f0f0;
      cursor: pointer;
      border-radius: 4px;
    }
    button.active {
      background: #007bff;
      color: white;
    }
    .paper-item {
      padding: 10px;
      border: 1px solid #ddd;
      margin-bottom: 10px;
      border-radius: 4px;
    }
    .paper-title {
      font-weight: bold;
      margin-bottom: 5px;
    }
    .paper-notes {
      font-size: 12px;
      color: #666;
      margin-top: 5px;
    }
  </style>
</head>
<body>
  <h2>Scholar Plus</h2>

  <div class="tab-buttons">
    <button class="tab-btn active" data-tab="saved">Saved Papers</button>
    <button class="tab-btn" data-tab="settings">Settings</button>
  </div>

  <div id="saved-papers"></div>
  <div id="settings" style="display: none;"></div>

>>>>>>> 957a05ec9ec85ac69b64fcda12b5f2b7f2d068ca
  <script src="popup.js"></script>
</body>
</html>
```

```javascript
// popup.js
<<<<<<< HEAD
document.getElementById('export-csv').addEventListener('click', async () => {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })

  const status = document.getElementById('status')
  status.textContent = 'Extracting results...'

  const response = await chrome.tabs.sendMessage(tab.id, { action: 'exportCsv' })

  if (response?.success) {
    status.textContent = 'CSV downloaded successfully'
  } else {
    status.textContent = 'Error: are you on a Scholar results page?'
  }
})

document.getElementById('copy-titles').addEventListener('click', async () => {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })
  const response = await chrome.tabs.sendMessage(tab.id, { action: 'scrapePage' })

  if (response?.papers) {
    const titles = response.papers.map(p => p.title).join('\n')
    await navigator.clipboard.writeText(titles)
    document.getElementById('status').textContent = `${response.papers.length} titles copied`
  }
})
```

### Background Service Worker

```javascript
// background.js
// Manifest V3 uses service workers instead of persistent background pages

chrome.runtime.onInstalled.addListener(() => {
  console.log('Scholar Research Assistant installed')
})

// Handle extension icon click — open Scholar if not on Scholar tab
chrome.action.onClicked.addListener(async (tab) => {
  if (!tab.url?.includes('scholar.google.com')) {
    await chrome.tabs.create({ url: 'https://scholar.google.com' })
  }
})
```

## Loading and Testing the Extension

```bash
# Build if using webpack
npm run build

# Manual loading for development:
# 1. Go to chrome://extensions
# 2. Enable "Developer mode" (top right toggle)
# 3. Click "Load unpacked"
# 4. Select your extension directory

# For production: package as .crx or submit to Chrome Web Store
```

## Handling Scholar's Dynamic Content

Google Scholar loads some content dynamically. If your content script runs before content renders, use a MutationObserver:

```javascript
// content.js — wait for results to load
function waitForResults(callback) {
  const existing = document.querySelectorAll('.gs_r.gs_or.gs_scl')
  if (existing.length > 0) {
    callback()
    return
  }

  const observer = new MutationObserver((mutations, obs) => {
    const results = document.querySelectorAll('.gs_r.gs_or.gs_scl')
    if (results.length > 0) {
      obs.disconnect()
      callback()
    }
  })

  observer.observe(document.body, { childList: true, subtree: true })
}

waitForResults(() => {
  // Safe to scrape now
  const papers = scrapeCurrentPage()
  console.log(`Found ${papers.length} papers`)
})
```

## Manifest V3 Migration Notes

If you're updating a Manifest V2 extension:
- Replace `background.page` or `background.scripts` with `background.service_worker`
- Replace `browser_action`/`page_action` with `action`
- Replace `chrome.browserAction` with `chrome.action`
- Service workers cannot use DOM APIs or persistent state — use `chrome.storage` instead of globals
=======
function loadSavedPapers() {
  chrome.storage.local.get('savedPapers', (result) => {
    const papers = result.savedPapers || [];
    const container = document.getElementById('saved-papers');

    if (papers.length === 0) {
      container.innerHTML = '<p>No saved papers yet</p>';
      return;
    }

    container.innerHTML = papers.map(paper => `
      <div class="paper-item">
        <div class="paper-title">${paper.title}</div>
        <div class="paper-notes">${paper.notes || 'No notes'}</div>
        <small>${new Date(paper.savedAt).toLocaleDateString()}</small>
        <button onclick="removePaper('${paper.id}')">Remove</button>
      </div>
    `).join('');
  });
}

function removePaper(paperId) {
  chrome.storage.local.get('savedPapers', (result) => {
    const papers = result.savedPapers || [];
    const filtered = papers.filter(p => p.id !== paperId);
    chrome.storage.local.set({ savedPapers: filtered });
    loadSavedPapers();
  });
}

// Load papers when popup opens
loadSavedPapers();
```

## Publishing Your Extension

1. **Create developer account** ($5 one-time, Chrome Web Store)
2. **Create extension ZIP**: Exclude `.git`, `node_modules`
3. **Upload to Chrome Web Store** with screenshots, description, privacy policy
4. **Review takes 1-3 days**
5. **Once approved**, appears in Chrome Web Store (anyone can install)

## Testing Before Publishing

```bash
# 1. Open Chrome Extensions page
# chrome://extensions/

# 2. Enable "Developer mode" (top right)

# 3. Click "Load unpacked"

# 4. Select your extension folder

# 5. Extension loads (any changes require refresh)

# 6. Test on Google Scholar (scholar.google.com)
```

## Common Pitfalls and Solutions

**Pitfall 1: Extension doesn't load**
- Manifest V3 syntax incorrect
- Invalid JSON in manifest.json

*Solution*: Validate manifest.json at jsonlint.com. Ensure all permissions are array.

**Pitfall 2: Content script doesn't run**
- host_permissions not set correctly
- Domain doesn't match `matches` pattern

*Solution*: Check `matches` field. Scholar URLs must be exact: `https://scholar.google.com/*`

**Pitfall 3: Storage data doesn't persist**
- Using session storage instead of chrome.storage
- Not handling async properly

*Solution*: Use `chrome.storage.local` (persists across sessions). All chrome APIs are async (use callbacks or promises).

**Pitfall 4: Extension slows down Scholar**
- Content script running inefficiently
- DOM mutations causing repeated queries

*Solution*: Use `requestAnimationFrame` to batch DOM updates. Cache selectors. Use MutationObserver sparingly.

## Real Workflow: Using Your Scholar Extension

1. **Morning**: Search Scholar for "machine learning papers 2025"
2. **Find interesting paper**: Click "Save" button
3. **Read paper later**: Open extension popup, see saved list
4. **Export for thesis**: Click "Cite" on saved paper, copy BibTeX
5. **Find free version**: Click "Find PDF", opens free version from unpaywall

**Time saved per paper**: 3-5 minutes (vs manual citation lookup, PDF search)

## Building Additional Features: Export to Zotero

Add button to export saved papers to Zotero (research management tool):

```javascript
// Add to content.js
async function exportToZotero(paperId, title) {
  const zoteroWebAPIKey = await getZoteroAPIKey();

  const response = await fetch('https://api.zotero.org/users/[userID]/items', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${zoteroWebAPIKey}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      items: [{
        itemType: 'journalArticle',
        title: title,
        url: `https://scholar.google.com/scholar?q=${paperId}`
      }]
    })
  });

  if (response.ok) {
    showNotification('Added to Zotero!');
  }
}
```

## Team Exercise: Planning Your Extension (60 minutes)

**Part 1: Needs (15 min)**
1. What annoys you most about Google Scholar?
2. What repetitive task do you do every time you search?
3. What would save you most time?

**Part 2: Feature Spec (20 min)**
1. Pick top 3 features
2. Sketch UI for each feature
3. How would each feature work? (Step by step)

**Part 3: Technical Design (15 min)**
1. Which data must persist? (saved papers, settings)
2. Need API access? (unpaywall, Zotero, etc.)
3. Which APIs require user authentication?

**Part 4: Roadmap (10 min)**
1. MVP (minimum viable): Just save papers + basic export
2. V1.1: Find free PDFs
3. V1.2: Export to Zotero
>>>>>>> 957a05ec9ec85ac69b64fcda12b5f2b7f2d068ca

## Frequently Asked Questions

**How long does it take to build this extension?**

<<<<<<< HEAD
For a straightforward setup, expect 30 minutes to 2 hours depending on your familiarity with the tools involved. Having your credentials and environment ready before starting saves significant time.

**Do I need prior experience to follow this guide?**

Basic familiarity with JavaScript and the command line is helpful. If you get stuck, the official Chrome Extensions documentation covers fundamentals.
=======
A basic version with export-to-CSV takes 2–4 hours for a developer comfortable with JavaScript. Adding saved search alerts and a polished popup UI adds another 3–5 hours. Plan for a full weekend if you want something production-ready.

**What are the most common mistakes to avoid?**

Using Manifest V2 patterns in a V3 extension is the most frequent issue — particularly using `chrome.browserAction` instead of `chrome.action`, or trying to use a persistent background page instead of a service worker. The Chrome migration guide covers every breaking change.

**Do I need prior experience to follow this guide?**

Basic JavaScript familiarity is required. You do not need to know browser extension APIs beforehand — the guide covers every API call used. If DOM manipulation and event listeners are familiar, you have enough background.
>>>>>>> 957a05ec9ec85ac69b64fcda12b5f2b7f2d068ca

**Can I adapt this for a different academic search engine?**

<<<<<<< HEAD
Yes, the underlying concepts transfer to Firefox extensions (WebExtensions API is similar). The manifest format differs slightly but content scripts and message passing work the same way.

## Related Articles

- [Chrome Extension Compress Images Before Upload](/remote-work-tools/chrome-extension-compress-images-before-upload/)
- [Chrome Extension Linear Issue Tracker: Practical Guide](/remote-work-tools/chrome-extension-linear-issue-tracker/)
- [Chrome Extension MLA Citation Generator: A Developer Guide](/remote-work-tools/chrome-extension-mla-citation-generator/)

=======
Yes. PubMed, Semantic Scholar, and arXiv all have parseable HTML. Change the `host_permissions` and `content_scripts` matches in the manifest, then update the CSS selectors in `parseResults()`. The background service worker and popup code remain largely the same.

**Where can I get help if I run into issues?**

The Chrome Developers documentation at developer.chrome.com is the authoritative source. The `chrome-extensions` tag on Stack Overflow is active for specific error messages. The Chromium extensions Google Group handles edge cases and API behavior questions.

## Related Articles

- [Chrome Extension Linear Issue Tracker: Practical Guide](/remote-work-tools/chrome-extension-linear-issue-tracker/)
- [How to Manage Cross-Functional Remote Projects](/remote-work-tools/how-to-manage-cross-functional-remote-projects/)
- [Chrome Extension OneNote Clipper Setup: Complete Guide](/remote-work-tools/chrome-extension-onenote-clipper-setup/)
- [Chrome Extension Compress Images Before Upload](/remote-work-tools/chrome-extension-compress-images-before-upload/)
- [Chrome Extension Currency Converter Shopping](/remote-work-tools/chrome-extension-currency-converter-shopping/)
>>>>>>> 957a05ec9ec85ac69b64fcda12b5f2b7f2d068ca
Built by theluckystrike — More at [zovo.one](https://zovo.one)
