# Browser Extension Documentation

## Overview

The SurfSense Browser Extension is a cross-browser extension built with Plasmo (Manifest V3) that enables users to save web pages directly to their SurfSense knowledge base, especially useful for content behind authentication.

**Location**: `surfsense_browser_extension/`

**Key Technology**: Plasmo Framework (React-based Manifest V3)

**Supported Browsers**:
- Chrome
- Firefox
- Edge
- Safari (with some limitations)
- Brave

## Why Use the Extension?

### Primary Use Cases

1. **Save Authenticated Content**
   - Internal company wikis
   - Subscription-based articles
   - Private documentation
   - Social media posts
   - Email threads (webmail)

2. **Quick Capture**
   - Save interesting articles while browsing
   - Capture research material on the fly
   - Build reading lists
   - Archive web pages

3. **Context Menu Integration**
   - Right-click any page → "Save to SurfSense"
   - Quick access without leaving your workflow

## Architecture

### Extension Components

```
surfsense_browser_extension/
├── popup.tsx              # Extension popup UI
├── content.ts             # Content script (runs on web pages)
├── background/            # Background service worker
│   ├── index.ts          # Main background script
│   └── messages/         # Message handlers
├── routes/               # React Router pages
│   ├── pages/           # Individual pages
│   │   ├── home.tsx
│   │   ├── login.tsx
│   │   └── settings.tsx
│   └── ui/              # UI components
├── lib/                 # Utilities and helpers
├── utils/               # Shared utilities
└── assets/              # Icons and images
```

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Web Page                             │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Content Script (content.ts)                       │  │
│  │                                                   │  │
│  │ - Runs on every web page                         │  │
│  │ - Can access DOM                                  │  │
│  │ - Extracts page content                           │  │
│  │ - Cannot directly call APIs (security)            │  │
│  └────────────┬──────────────────────────────────────┘  │
└───────────────┼──────────────────────────────────────────┘
                │
                │ Message Passing
                ▼
┌─────────────────────────────────────────────────────────┐
│        Background Service Worker (background/)          │
│  ┌───────────────────────────────────────────────────┐  │
│  │ - Persistent background script                    │  │
│  │ - Handles API calls                               │  │
│  │ - Manages extension state                         │  │
│  │ - Processes messages from content/popup           │  │
│  └────────────┬──────────────────────────────────────┘  │
└───────────────┼──────────────────────────────────────────┘
                │
                │ Chrome Extension APIs
                ├──────────────────────┐
                │                      │
                ▼                      ▼
┌──────────────────────┐   ┌──────────────────────┐
│   Popup UI           │   │  SurfSense Backend   │
│   (popup.tsx)        │   │                      │
│                      │   │  POST /api/v1/       │
│  - React app         │   │  documents/          │
│  - User interaction  │   │                      │
│  - Settings          │   │  {                   │
│  - Login             │   │    document_type:    │
└──────────────────────┘   │      "EXTENSION",    │
                           │    content: [...]     │
                           │  }                   │
                           └──────────────────────┘
```

## Installation

### For Development

1. **Install dependencies**:
```bash
cd surfsense_browser_extension
pnpm install
# or
npm install
```

2. **Set environment variables** (`.env`):
```bash
PLASMO_PUBLIC_BACKEND_URL=http://localhost:8000
```

3. **Run development build**:
```bash
pnpm dev
# or
npm run dev
```

This creates `build/chrome-mv3-dev/` directory.

4. **Load in browser**:

**Chrome/Edge/Brave**:
1. Navigate to `chrome://extensions/`
2. Enable "Developer mode"
3. Click "Load unpacked"
4. Select `build/chrome-mv3-dev/` directory

**Firefox**:
1. Navigate to `about:debugging#/runtime/this-firefox`
2. Click "Load Temporary Add-on"
3. Select `manifest.json` from `build/firefox-mv3-dev/`

### For Production

1. **Build extension**:
```bash
pnpm build
# or
npm run build
```

This creates production builds for all browsers in `build/` directory.

2. **Package for distribution**:
```bash
# Zip the appropriate build folder
cd build/chrome-mv3-prod
zip -r ../../surfsense-chrome.zip .
```

3. **Submit to stores**:
- Chrome Web Store
- Firefox Add-ons
- Edge Add-ons
- (See Plasmo documentation for automated submission)

## Features

### 1. Authentication

**Login Flow**:
```typescript
// User logs in via popup
1. User enters credentials
2. Extension calls /auth/jwt/login
3. Receives JWT token
4. Stores token in chrome.storage.local
5. Token used for all subsequent API calls
```

**Token Storage**:
```typescript
// lib/storage.ts
import { Storage } from "@plasmohq/storage"

const storage = new Storage()

export const setToken = async (token: string) => {
  await storage.set("surfsense_token", token)
}

export const getToken = async () => {
  return await storage.get("surfsense_token")
}
```

### 2. Saving Pages

**User Flow**:
1. User browses to a page
2. Clicks extension icon
3. Selects search space
4. Clicks "Save Page"
5. Extension captures page content
6. Sends to backend
7. Shows success/error notification

**Content Extraction**:
```typescript
// content.ts
export {}

// Listen for messages from popup/background
chrome.runtime.onMessage.addListener((request, sender, sendResponse) => {
  if (request.action === "getPageContent") {
    const content = {
      title: document.title,
      url: window.location.href,
      html: document.documentElement.outerHTML,
      text: document.body.innerText,
      metadata: {
        author: getMetaContent("author"),
        description: getMetaContent("description"),
        keywords: getMetaContent("keywords"),
      }
    }
    sendResponse(content)
  }
})

function getMetaContent(name: string): string {
  const meta = document.querySelector(`meta[name="${name}"]`)
  return meta?.getAttribute("content") || ""
}
```

**Sending to Backend**:
```typescript
// background/messages/savePage.ts
import type { PlasmoMessaging } from "@plasmohq/messaging"

const handler: PlasmoMessaging.MessageHandler = async (req, res) => {
  const { content, searchSpaceId } = req.body
  
  try {
    const token = await getToken()
    
    const response = await fetch(
      `${process.env.PLASMO_PUBLIC_BACKEND_URL}/api/v1/documents/`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${token}`,
        },
        body: JSON.stringify({
          search_space_id: searchSpaceId,
          document_type: "EXTENSION",
          content: [{
            title: content.title,
            url: content.url,
            content: content.text,
            metadata: content.metadata,
          }],
        }),
      }
    )

    if (!response.ok) {
      throw new Error("Failed to save page")
    }

    res.send({ success: true })
  } catch (error) {
    res.send({ success: false, error: error.message })
  }
}

export default handler
```

### 3. Context Menu

**Registration**:
```typescript
// background/index.ts
chrome.runtime.onInstalled.addListener(() => {
  chrome.contextMenus.create({
    id: "savePage",
    title: "Save to SurfSense",
    contexts: ["page"],
  })
})

chrome.contextMenus.onClicked.addListener((info, tab) => {
  if (info.menuItemId === "savePage") {
    // Send message to content script to get page data
    chrome.tabs.sendMessage(tab.id, { action: "savePage" })
  }
})
```

### 4. Search Spaces Selection

**Fetch User's Search Spaces**:
```typescript
// routes/pages/home.tsx
import { useEffect, useState } from "react"

export function HomePage() {
  const [searchSpaces, setSearchSpaces] = useState([])
  const [selectedSpace, setSelectedSpace] = useState(null)

  useEffect(() => {
    fetchSearchSpaces()
  }, [])

  const fetchSearchSpaces = async () => {
    const token = await getToken()
    const response = await fetch(
      `${process.env.PLASMO_PUBLIC_BACKEND_URL}/api/v1/searchspaces/`,
      {
        headers: {
          Authorization: `Bearer ${token}`,
        },
      }
    )
    const data = await response.json()
    setSearchSpaces(data)
  }

  const handleSavePage = async () => {
    // Get current tab
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })
    
    // Send message to content script
    const response = await chrome.tabs.sendMessage(tab.id, {
      action: "getPageContent",
    })

    // Save via background script
    await chrome.runtime.sendMessage({
      name: "savePage",
      body: {
        content: response,
        searchSpaceId: selectedSpace,
      },
    })
  }

  return (
    <div>
      <select
        value={selectedSpace}
        onChange={(e) => setSelectedSpace(Number(e.target.value))}
      >
        {searchSpaces.map((space) => (
          <option key={space.id} value={space.id}>
            {space.name}
          </option>
        ))}
      </select>
      <button onClick={handleSavePage}>Save Page</button>
    </div>
  )
}
```

## Popup UI

### Pages

**Home** (`routes/pages/home.tsx`):
- Display current page info
- Select search space
- Save page button
- Recent saves

**Login** (`routes/pages/login.tsx`):
- Email/password input
- Login button
- Link to register

**Settings** (`routes/pages/settings.tsx`):
- Backend URL configuration
- Auto-save preferences
- Keyboard shortcuts
- Logout button

### Routing

```typescript
// routes/index.tsx
import { Route, Routes } from "react-router-dom"
import { HomePage } from "./pages/home"
import { LoginPage } from "./pages/login"
import { SettingsPage } from "./pages/settings"

export function Routing() {
  return (
    <Routes>
      <Route path="/" element={<HomePage />} />
      <Route path="/login" element={<LoginPage />} />
      <Route path="/settings" element={<SettingsPage />} />
    </Routes>
  )
}
```

## Permissions

Extension requires these permissions:

```json
// package.json (manifest configuration)
{
  "manifest": {
    "permissions": [
      "storage",        // Store authentication token
      "activeTab",      // Access current tab
      "contextMenus",   // Right-click menu
      "tabs"            // Query tabs
    ],
    "host_permissions": [
      "<all_urls>"      // Access content on all sites
    ]
  }
}
```

**Why each permission?**:
- `storage`: Store JWT token and user preferences
- `activeTab`: Read content from the page user is viewing
- `contextMenus`: Add "Save to SurfSense" to right-click menu
- `tabs`: Get information about current tab
- `<all_urls>`: Content script needs to run on any page

## Security Considerations

### Token Storage

✅ **Do**:
- Store token in `chrome.storage.local` (encrypted by browser)
- Clear token on logout
- Refresh token periodically

❌ **Don't**:
- Store token in localStorage (accessible by web pages)
- Log token to console
- Include token in URLs

### Content Access

✅ **Do**:
- Only extract content when user explicitly requests
- Respect robots.txt and site policies
- Handle sensitive data appropriately

❌ **Don't**:
- Automatically save pages without user action
- Send content to third-party servers
- Extract password fields or payment info

### API Communication

✅ **Do**:
- Use HTTPS for all API calls
- Validate SSL certificates
- Handle API errors gracefully

❌ **Don't**:
- Trust all certificates (no `--insecure` flags)
- Expose API keys in code
- Store sensitive data unencrypted

## Testing

### Manual Testing

1. **Install development build**
2. **Test authentication flow**:
   - Login with valid credentials
   - Login with invalid credentials
   - Logout
   - Token persistence across browser restarts

3. **Test page saving**:
   - Public pages
   - Authenticated pages (e.g., Twitter after login)
   - Pages with special characters
   - Large pages

4. **Test context menu**:
   - Right-click on page
   - Verify menu item appears
   - Click to save

5. **Test different browsers**:
   - Chrome
   - Firefox
   - Edge

### Automated Testing

Currently minimal. Recommended structure:

```
tests/
├── unit/
│   ├── storage.test.ts
│   └── content.test.ts
└── e2e/
    ├── login.test.ts
    └── savePage.test.ts
```

## Debugging

### Chrome DevTools

**Popup**:
1. Right-click extension icon
2. Click "Inspect popup"
3. DevTools opens for popup

**Background Script**:
1. Go to `chrome://extensions/`
2. Find SurfSense extension
3. Click "service worker" link
4. DevTools opens for background script

**Content Script**:
1. Open DevTools on any web page (F12)
2. Content script logs appear in page's console
3. Look for `[SurfSense]` prefix

### Firefox Debugging

1. Navigate to `about:debugging#/runtime/this-firefox`
2. Find SurfSense extension
3. Click "Inspect"
4. Console shows all logs (popup, background, content)

### Common Issues

**Issue**: Extension doesn't load
**Solution**: Check manifest.json syntax, ensure all files exist

**Issue**: Content script not running
**Solution**: Check host_permissions, reload extension and page

**Issue**: API calls fail
**Solution**: Check CORS settings on backend, verify token is valid

**Issue**: Token not persisting
**Solution**: Use chrome.storage.local, not localStorage

## Distribution

### Chrome Web Store

1. Create developer account ($5 one-time fee)
2. Prepare store listing:
   - Icon (128x128)
   - Screenshots
   - Description
   - Privacy policy
3. Upload ZIP file
4. Submit for review
5. Usually approved within 1-3 days

### Firefox Add-ons

1. Create developer account (free)
2. Prepare listing (similar to Chrome)
3. Upload ZIP or XPI file
4. Submit for review
5. Usually approved within 1-3 days

### Edge Add-ons

1. Create developer account (free)
2. Can use Chrome Web Store package
3. Upload and submit
4. Usually approved within 1-2 days

## Future Enhancements

Potential features to add:

- [ ] Automatic tagging based on page content
- [ ] Bulk save (save all tabs)
- [ ] Keyboard shortcuts (Ctrl+Shift+S)
- [ ] Annotation while saving (add notes)
- [ ] PDF export before saving
- [ ] Screenshot capture
- [ ] Video transcript extraction
- [ ] Sync across devices
- [ ] Offline queue (save when offline, sync later)
- [ ] Smart recommendations (suggest related saved pages)

## Development Tips

### Hot Reload

Plasmo supports hot reload for most changes:
- Popup UI: Auto-reloads
- Content scripts: Refresh page
- Background: Refresh extension

### Code Organization

```typescript
// Use consistent patterns
lib/       → Pure utilities (no Chrome APIs)
utils/     → Extension-specific utilities (use Chrome APIs)
routes/    → UI components and pages
background/→ Background logic only
```

### State Management

For complex state, consider:
- Zustand (lightweight)
- Redux (if needed)
- Context API (built-in React)

### Styling

Extension uses Tailwind CSS:
```typescript
<button className="px-4 py-2 bg-blue-500 text-white rounded">
  Save Page
</button>
```

## Resources

- **Plasmo Documentation**: https://docs.plasmo.com/
- **Chrome Extension Docs**: https://developer.chrome.com/docs/extensions/
- **Firefox Extension Docs**: https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions
- **Manifest V3 Migration**: https://developer.chrome.com/docs/extensions/mv3/intro/

## Contributing

See main [Contributing Guide](../../CONTRIBUTING.md) for general guidelines.

**Extension-specific**:
1. Follow Plasmo best practices
2. Test on all supported browsers
3. Ensure backward compatibility
4. Update documentation for new features
5. Add tests for critical paths

## Troubleshooting

**Issue**: Build fails
**Solution**: Clear `.plasmo` folder, reinstall dependencies

**Issue**: Extension not updating
**Solution**: Click reload button in `chrome://extensions/`, hard refresh pages

**Issue**: Storage not working
**Solution**: Check permissions in manifest, use `chrome.storage` not `localStorage`

**Issue**: Content script conflicts with website
**Solution**: Use unique class names, namespace CSS, avoid global variables

## Summary

The SurfSense Browser Extension:
- ✅ Saves web pages to SurfSense
- ✅ Handles authenticated content
- ✅ Cross-browser compatible (Manifest V3)
- ✅ Secure token storage
- ✅ Context menu integration
- ✅ React-based UI with Plasmo
- ✅ Easy to develop and test

For more details, see:
- [Architecture Documentation](../ARCHITECTURE.md)
- [API Reference](../API_REFERENCE.md)
- [Getting Started Guide](../GETTING_STARTED.md)
