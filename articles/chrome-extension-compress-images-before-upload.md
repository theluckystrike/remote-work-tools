---
layout: default
title: "Chrome Extension Compress Images Before Upload: A."
description: "Learn how to build a Chrome extension that automatically compresses images before upload. Perfect for developers and power users who need to optimize."
date: 2026-03-15
author: theluckystrike
permalink: /chrome-extension-compress-images-before-upload/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
---

{% raw %}

# Chrome Extension Compress Images Before Upload: A Practical Guide

Image file sizes can quickly become a bottleneck when uploading content to websites, content management systems, or cloud storage. A 10MB photo from your smartphone might be perfect for printing, but it creates slow upload times, consumes unnecessary bandwidth, and can hit file size limits on many platforms. Building a Chrome extension to compress images before upload solves this problem directly in your browser.

This guide walks you through creating a Chrome extension that intercepts file inputs, compresses images using the Canvas API, and replaces the original file with an optimized version. You'll have a working solution that works across any website with file upload functionality.

## Why Client-Side Compression Matters

Uploading unoptimized images affects both performance and user experience. Large images take longer to upload, especially on slower connections. Many platforms impose strict file size limits—WordPress defaults to 2MB, email services often cap attachments at 25MB, and API endpoints may reject payloads exceeding certain thresholds.

Client-side compression using the Canvas API offers several advantages. The compression happens locally on the user's device, meaning no server-side processing is required. This reduces bandwidth usage and upload times significantly. The entire process happens in the browser, keeping data private and eliminating the need for external compression services.

## Setting Up Your Extension Structure

Every Chrome extension requires a manifest file and a background service worker. For this image compression extension, you'll need:

```
/compress-before-upload
  /manifest.json
  /content.js
  /background.js
  /popup.html
  /popup.js
  /icon.png
```

The manifest defines permissions and registers the extension's components. Your content script will handle detecting file input changes, while the background script manages communication between components.

## Writing the Manifest

Create a `manifest.json` file with the necessary permissions:

```json
{
  "manifest_version": 3,
  "name": "Compress Images Before Upload",
  "version": "1.0",
  "description": "Automatically compress images before uploading to any website",
  "permissions": ["storage", "activeTab", "scripting"],
  "host_permissions": ["<all_urls>"],
  "action": {
    "default_popup": "popup.html",
    "default_icon": "icon.png"
  },
  "content_scripts": [{
    "matches": ["<all_urls>"],
    "js": ["content.js"]
  }]
}
```

Manifest V3 requires service workers instead of background pages. The `host_permissions` field grants access to all URLs, which is necessary for the extension to work across different websites.

## Implementing the Compression Logic

The core compression happens in your content script. This script detects file input elements and monitors them for changes:

```javascript
// content.js
class ImageCompressor {
  constructor() {
    this.quality = 0.7;
    this.maxWidth = 1920;
    this.maxHeight = 1920;
    this.init();
  }

  init() {
    this.observeFileInputs();
    this.setupMutationObserver();
  }

  observeFileInputs() {
    document.querySelectorAll('input[type="file"]').forEach(input => {
      if (input.accept && input.accept.includes('image')) {
        input.addEventListener('change', (e) => this.handleFileSelect(e));
      }
    });
  }

  setupMutationObserver() {
    const observer = new MutationObserver(mutations => {
      mutations.forEach(mutation => {
        mutation.addedNodes.forEach(node => {
          if (node.nodeType === Node.ELEMENT_NODE) {
            const inputs = node.querySelectorAll ? 
              node.querySelectorAll('input[type="file"]') : [];
            inputs.forEach(input => {
              if (input.accept && input.accept.includes('image')) {
                input.addEventListener('change', (e) => this.handleFileSelect(e));
              }
            });
          }
        });
      });
    });

    observer.observe(document.body, { childList: true, subtree: true });
  }

  async handleFileSelect(event) {
    const input = event.target;
    const files = Array.from(input.files);
    
    for (const file of files) {
      if (file.type.startsWith('image/')) {
        const compressedFile = await this.compressImage(file);
        this.replaceFile(input, compressedFile);
      }
    }
  }

  async compressImage(file) {
    return new Promise((resolve) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
          const canvas = document.createElement('canvas');
          let { width, height } = img;

          // Scale down if needed
          if (width > this.maxWidth || height > this.maxHeight) {
            const ratio = Math.min(
              this.maxWidth / width,
              this.maxHeight / height
            );
            width *= ratio;
            height *= ratio;
          }

          canvas.width = width;
          canvas.height = height;
          
          const ctx = canvas.getContext('2d');
          ctx.drawImage(img, 0, 0, width, height);

          canvas.toBlob(
            (blob) => {
              const compressedFile = new File([blob], file.name, {
                type: 'image/jpeg',
                lastModified: Date.now()
              });
              resolve(compressedFile);
            },
            'image/jpeg',
            this.quality
          );
        };
        img.src = e.target.result;
      };
      reader.readAsDataURL(file);
    });
  }

  replaceFile(input, newFile) {
    const dataTransfer = new DataTransfer();
    dataTransfer.items.add(newFile);
    input.files = dataTransfer.files;
    
    // Trigger change event for React/Angular form handling
    input.dispatchEvent(new Event('change', { bubbles: true }));
  }
}

new ImageCompressor();
```

This content script automatically attaches to file input elements that accept images. When a user selects files, it compresses each image using the Canvas API and replaces the original file with the compressed version.

## Adding User Controls

Users should be able to adjust compression settings. Create a simple popup interface:

```html
<!-- popup.html -->
<!DOCTYPE html>
<html>
<head>
  <style>
    body { width: 280px; padding: 16px; font-family: system-ui; }
    label { display: block; margin-bottom: 8px; font-weight: 500; }
    input[type="range"] { width: 100%; margin-bottom: 16px; }
    .value-display { float: right; color: #666; }
    .info { font-size: 12px; color: #666; margin-top: 12px; }
  </style>
</head>
<body>
  <h3>Image Compression</h3>
  
  <label>
    Quality: <span id="qualityValue" class="value-display">70%</span>
  </label>
  <input type="range" id="quality" min="0.1" max="1" step="0.1" value="0.7">
  
  <label>
    Max Width: <span id="widthValue" class="value-display">1920px</span>
  </label>
  <input type="range" id="maxWidth" min="800" max="4096" step="100" value="1920">
  
  <label>
    Max Height: <span id="heightValue" class="value-display">1920px</span>
  </label>
  <input type="range" id="maxHeight" min="600" max="4096" step="100" value="1920">
  
  <p class="info">Changes apply to next file selection.</p>
  
  <script src="popup.js"></script>
</body>
</html>
```

The popup script saves settings to Chrome storage:

```javascript
// popup.js
document.addEventListener('DOMContentLoaded', () => {
  // Load saved settings
  chrome.storage.sync.get(['quality', 'maxWidth', 'maxHeight'], (settings) => {
    if (settings.quality) {
      document.getElementById('quality').value = settings.quality;
      document.getElementById('qualityValue').textContent = 
        Math.round(settings.quality * 100) + '%';
    }
    if (settings.maxWidth) {
      document.getElementById('maxWidth').value = settings.maxWidth;
      document.getElementById('widthValue').textContent = settings.maxWidth + 'px';
    }
    if (settings.maxHeight) {
      document.getElementById('maxHeight').value = settings.maxHeight;
      document.getElementById('heightValue').textContent = settings.maxHeight + 'px';
    }
  });

  // Save settings on change
  document.getElementById('quality').addEventListener('input', (e) => {
    const value = parseFloat(e.target.value);
    document.getElementById('qualityValue').textContent = Math.round(value * 100) + '%';
    chrome.storage.sync.set({ quality: value });
  });

  document.getElementById('maxWidth').addEventListener('input', (e) => {
    const value = parseInt(e.target.value);
    document.getElementById('widthValue').textContent = value + 'px';
    chrome.storage.sync.set({ maxWidth: value });
  });

  document.getElementById('maxHeight').addEventListener('input', (e) => {
    const value = parseInt(e.target.value);
    document.getElementById('heightValue').textContent = value + 'px';
    chrome.storage.sync.set({ maxHeight: value });
  });
});
```

Update the content script to read these settings from storage before compressing:

```javascript
async loadSettings() {
  return new Promise((resolve) => {
    chrome.storage.sync.get(['quality', 'maxWidth', 'maxHeight'], (settings) => {
      this.quality = settings.quality || 0.7;
      this.maxWidth = settings.maxWidth || 1920;
      this.maxHeight = settings.maxHeight || 1920;
      resolve();
    });
  });
}
```

## Testing Your Extension

Load your extension in Chrome by following these steps:

1. Navigate to `chrome://extensions/`
2. Enable "Developer mode" in the top right corner
3. Click "Load unpacked" and select your extension directory
4. Visit any website with file upload functionality
5. Select an image file and verify the compression works

The extension will automatically compress images when you select them through file input elements. You can adjust the quality settings using the extension popup.

## Limitations and Considerations

This approach works well for most use cases but has some constraints. The Canvas API outputs JPEG format, so PNG images with transparency will lose their alpha channel. If transparency is essential, consider using WebP output or preserving the original format for PNG files.

Very large images might cause memory issues on lower-end devices. The extension includes dimension limits to help prevent this, but you can adjust these based on your typical use case.

Some web applications use custom file upload components that don't use standard `<input type="file">` elements. In these cases, you'll need to extend the content script to handle their specific upload mechanisms.

## Conclusion

Building a Chrome extension for image compression before upload gives you control over file sizes without relying on external services. The extension works automatically in the background, compressing images as you select them for upload. With adjustable quality and dimension settings, you can balance file size against image quality based on your specific needs.

The implementation uses only browser-native APIs, keeping the extension lightweight and private. All processing happens locally on your device, making it fast and secure.


## Related Reading

- [Remote Work Guides Hub](/remote-work-tools/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
