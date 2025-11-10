# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Development Commands

### Loading and Testing the Extension

```bash
# The extension is loaded manually via chrome://extensions/ or edge://extensions/
# After code changes, click the refresh icon on the extension card in the browser
# Test by navigating to a Facebook Marketplace listing
```

### Key Files to Monitor

- Changes to manifest.json require full reload
- Content script changes (facebook-scraper.js) require page refresh
- Background script changes require extension reload
- Popup changes (HTML/CSS/JS) only need popup to be reopened

## Architecture Overview

### Extension Architecture (Manifest V3)

This is a Chrome extension that analyzes Facebook Marketplace listings by comparing them with resale prices from eBay and other platforms. It uses **Manifest V3** architecture with three main components that communicate via message passing:

1. **Content Script** (facebook-scraper.js)
   - Injected into facebook.com/marketplace/* pages
   - Extracts item data (title, price, condition, images, etc.) from the DOM
   - Automatically runs on page load and sends extracted data to background script
   - Uses heuristic selectors to handle Facebook's dynamic DOM

2. **Background Service Worker** (background/)
   - Acts as the central coordinator between all components
   - Imports and orchestrates api-client.js and rating-engine.js
   - Handles message routing and manages Chrome Storage API
   - Automatically triggers analysis when items are extracted (if configured)
   - Updates extension badge with rating scores

3. **Popup UI** (popup/)
   - User interface displayed when clicking the extension icon
   - Shows analysis results, comparable listings, and price insights
   - Provides settings panel for eBay API credentials
   - Polls for analysis completion via message passing

### Data Flow

```
Facebook Page → Content Script extracts item data
    ↓
Background receives 'itemExtracted' → stores in chrome.storage.local
    ↓
Background calls api-client.js → searches eBay API
    ↓
Background calls rating-engine.js → calculates score (0-100)
    ↓
Results stored in chrome.storage.local + badge updated
    ↓
Popup displays results by reading from storage
```

### Key Components

**api-client.js**
- Handles eBay Finding API integration (XML-based)
- Builds search queries from extracted item data
- Cleans keywords by removing condition terms
- Returns comparable listings with prices, conditions, URLs
- Stubbed methods for Mercari/Poshmark (require backend for CORS)

**rating-engine.js**
- Implements weighted scoring algorithm:
  - Price Match (40%): How good is the deal vs market average
  - Condition Match (25%): Item condition comparison
  - Market Demand (20%): Based on number of listings found
  - Price Variation (15%): Consistency of pricing data
- Generates confidence levels (high/medium/low)
- Provides buying recommendations and profit analysis

**facebook-scraper.js**
- DOM extraction using multiple fallback selectors
- Handles Facebook's frequently-changing markup
- Auto-runs 2 seconds after page load to allow dynamic content
- Extracts: title, price, description, images, location, condition, category

### Storage Strategy

- **chrome.storage.sync**: User settings (eBay API key, auto-analyze preference)
- **chrome.storage.local**: Current item data, analysis results, temporary state
- Analysis results include: rating, comparables (top 10), timestamp, source counts

### Rating Scale

- 85-100: Excellent Deal 🟢
- 70-84: Good Deal 🟡  
- 55-69: Fair Deal 🟠
- 40-54: Below Average 🔴
- 0-39: Poor Deal ❌

## Development Notes

### API Requirements

- eBay App ID required from https://developer.ebay.com
- Free tier has rate limits
- No backend required for current eBay integration (uses direct JSONP/XML API)

### Facebook Scraper Maintenance

Facebook frequently updates their DOM structure. If extraction fails:
1. Check browser console for errors in content script
2. Inspect Facebook's current markup for the target elements
3. Update selectors in facebook-scraper.js extraction methods
4. Test with multiple listing types (different categories, conditions)

### Adding New Resale Platforms

To integrate additional platforms (Mercari, Poshmark, etc.):
1. Add new search method in api-client.js
2. Update getComparableListings() to include the new source
3. Handle CORS issues (may require proxy or backend service)
4. Update background.js to merge results from new source
5. Ensure returned objects match the expected format: {title, price, condition, url, imageUrl, source}

### Message Passing

All component communication uses chrome.runtime.sendMessage/onMessage:
- Actions: 'itemExtracted', 'analyzeItem', 'getAnalysis', 'saveSettings'
- Always return true for async responses
- Use sendResponse() callback for returning data

### UI State Management

The popup has four states:
- loading: Analyzing item
- error: Analysis failed
- noData: Not on a marketplace listing
- results: Show analysis

State is controlled by showState() function which toggles visibility of state panels.

### Testing Workflow

1. Make code changes
2. Go to chrome://extensions/
3. Click refresh button on extension card
4. Navigate to https://www.facebook.com/marketplace/item/[any-listing-id]
5. Wait 2 seconds for auto-extraction
6. Click extension icon to view popup
7. Check browser console for any errors (F12)

### Common Issues

- **"No comparable listings found"**: Item is too unique, or eBay API quota exceeded
- **Price extraction fails**: Facebook updated markup, check console logs
- **Extension not auto-analyzing**: Check that eBay API key is configured in settings
- **Badge not updating**: Check background service worker console in chrome://extensions/
