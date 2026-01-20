# Amplitude Guides & Surveys with Segment Integration Guide

> **Implementation Guide for Third-Party Analytics Providers**

This guide documents how to integrate Amplitude Guides and Surveys when using Segment as your primary analytics provider (instead of Amplitude Browser SDK directly). In general, please refer to Amplitude's official documentation on how to instrument Guides and Surveys here: https://amplitude.com/docs/guides-and-surveys/sdk. This brief user guide has been put together for self learning purposes, it is tailored to a very specific use case, it refers to a very precise and simple architecture that constitutes a static webpage. Additionally, this guide is not constantly updated.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Event Forwarding](#event-forwarding)
6. [Debugging](#debugging)
7. [Common Issues](#common-issues)
8. [Production Checklist](#production-checklist)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Your Website                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐     ┌────────────────────┐                   │
│   │   Segment    │────▶│  Amplitude Actions │                   │
│   │  Analytics   │     │   (Destination)    │                   │
│   └──────────────┘     └────────────────────┘                   │
│          │                                                       │
│          │ anonymousId                                           │
│          ▼                                                       │
│   ┌──────────────────────────────────────────────┐              │
│   │     Amplitude Guides & Surveys SDK           │              │
│   │     (Uses same Device ID from Segment)       │              │
│   └──────────────────────────────────────────────┘              │
│          │                                                       │
│          │ forwardEvent()                                        │
│          ▼                                                       │
│   ┌──────────────────────────────────────────────┐              │
│   │  Event-based Guide/Survey Triggers           │              │
│   └──────────────────────────────────────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Device ID Sync**: Guides & Surveys MUST use the same Device ID as Segment (Segment's `anonymousId`)
2. **No `init()` needed**: Script loader auto-initializes; only `boot()` is required
3. **Bidirectional Event Flow**:
   - **G&S → Segment**: Survey events forwarded via `integrations` option
   - **Segment → G&S**: Track/Page events forwarded via `forwardEvent()` for triggers

---

## Prerequisites

- Segment Analytics account with write key
- Amplitude account with API key
- Guides & Surveys enabled in your Amplitude project
- GDPR consent mechanism (optional but recommended)

---

## Installation

### Step 1: Add SDK Scripts

Add the scripts in this order in your HTML:

```html
<!-- 1. Segment Snippet (standard installation) -->
<script>
    !function(){var i="analytics",analytics=window[i]=window[i]||[];
    // ... standard Segment snippet ...
    analytics._writeKey="YOUR_SEGMENT_WRITE_KEY";
    }}();
</script>

<!-- 2. Amplitude Session Replay (optional) -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>

<!-- 3. Amplitude Guides and Surveys SDK -->
<script src="https://cdn.amplitude.com/script/YOUR_AMPLITUDE_API_KEY.engagement.js"></script>
```

> **Important**: The Guides & Surveys script uses the **script loader** format which includes your API key in the URL. This auto-initializes the SDK.

---

## Configuration

### Step 2: Initialize After Consent (GDPR Compliant)

```javascript
const AMPLITUDE_API_KEY = 'YOUR_AMPLITUDE_API_KEY';

async function initializeAnalytics() {
    // Load Segment
    analytics.load('YOUR_SEGMENT_WRITE_KEY');
    analytics.identify(); // Create anonymousId
    
    analytics.ready(async function() {
        // Wait for Segment to be fully ready
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        // CRITICAL: Get Device ID from Segment
        let deviceId = null;
        
        // Method 1: From Segment's user object
        if (analytics.user) {
            deviceId = analytics.user().anonymousId();
        }
        
        // Method 2: Fallback to localStorage
        if (!deviceId) {
            const segmentAnonId = localStorage.getItem('ajs_anonymous_id');
            if (segmentAnonId) {
                deviceId = segmentAnonId.replace(/"/g, '');
            }
        }
        
        if (!deviceId) {
            console.error('Could not retrieve Segment device ID');
            return;
        }
        
        // Initialize Guides & Surveys
        await initializeGuidesAndSurveys(deviceId);
    });
}
```

### Step 3: Boot Guides & Surveys

```javascript
async function initializeGuidesAndSurveys(deviceId) {
    if (!window.engagement) {
        console.error('Guides & Surveys SDK not loaded');
        return;
    }
    
    // Boot with Segment's device ID
    // Note: init() is NOT needed - script loader auto-initializes
    await window.engagement.boot({
        user: {
            device_id: deviceId  // MUST match Segment's anonymousId
        },
        // Forward G&S events to Segment
        integrations: [{
            track: (event) => {
                analytics.track(event.event_type, event.event_properties);
            }
        }]
    });
    
    console.log('Guides & Surveys booted with device ID:', deviceId);
}
```

---

## Event Forwarding

### Segment → Guides & Surveys

To enable guides/surveys to trigger on Segment events, forward events using `forwardEvent()`:

```javascript
// Forward track events
analytics.on('track', (event, properties) => {
    if (window.engagement && window.engagement.forwardEvent) {
        window.engagement.forwardEvent({
            event_type: event,
            event_properties: properties || {}
        });
    }
});

// Forward page events
analytics.on('page', (category, name, properties) => {
    if (window.engagement && window.engagement.forwardEvent) {
        window.engagement.forwardEvent({
            event_type: '[Amplitude] Page Viewed',
            event_properties: {
                ...properties,
                page_name: name || 'Unknown',
                page_category: category || 'Unknown'
            }
        });
    }
});
```

### G&S → Segment

G&S events are automatically forwarded via the `integrations` option in `boot()`:

```javascript
integrations: [{
    track: (event) => {
        analytics.track(event.event_type, event.event_properties);
    }
}]
```

**Events forwarded include:**
- `Survey Viewed`
- `Survey Response Submitted`
- `Survey Dismissed`
- `Guide Viewed`
- `Guide Step Viewed`
- `Guide Dismissed`
- `Guide Completed`

---

## Debugging

### Check G&S Status

Run in browser console:

```javascript
window.engagement._debugStatus()
```

**Expected output:**
```javascript
{
    stateInitialized: true,
    decideSuccessful: true,
    num_guides_surveys: 2  // Number of active guides/surveys
}
```

### Verify Device ID Match

```javascript
// Segment's Device ID
analytics.user().anonymousId()

// Should match G&S Device ID (check network requests to gs.amplitude.com)
```

### Console Logs

The implementation logs helpful messages:

```
✅ Guides and Surveys SDK loaded (auto-initialized by script loader)
✅ Guides and Surveys booted successfully!
   Device ID: 926295ac-0a2c-4871-bbd3-08eae042b7f0
🎯 Guides and Surveys ready!
   - G&S events → Segment (via integrations)
   - Segment events → G&S (via forwardEvent)
🔄 Event forwarded to G&S: Analytics Initialized
🔄 Page event forwarded to G&S: Homepage
```

---

## Common Issues

### 1. "Engagement SDK has already been initialized"

**Cause**: Calling `init()` when using script loader

**Solution**: Don't call `init()` - the script loader auto-initializes. Only call `boot()`.

```javascript
// ❌ Don't do this with script loader
window.engagement.init(API_KEY, options);

// ✅ Just call boot()
await window.engagement.boot({ user: { device_id: deviceId } });
```

### 2. Guides/Surveys Not Appearing

**Possible causes:**
- Device ID mismatch between Segment and G&S
- `boot()` not called
- No active guides/surveys in Amplitude dashboard
- Targeting rules not met

**Debug steps:**
1. Check `window.engagement._debugStatus()`
2. Verify `decideSuccessful: true`
3. Confirm `num_guides_surveys > 0`

### 3. Event-Based Triggers Not Working

**Cause**: Events not forwarded to G&S

**Solution**: Ensure `forwardEvent()` is set up:

```javascript
analytics.on('track', (event, properties) => {
    window.engagement.forwardEvent({
        event_type: event,
        event_properties: properties
    });
});
```

### 4. G&S Events Not in Segment/Amplitude

**Cause**: Missing `integrations` in boot

**Solution**: Add track function to integrations:

```javascript
await window.engagement.boot({
    user: { device_id: deviceId },
    integrations: [{
        track: (event) => {
            analytics.track(event.event_type, event.event_properties);
        }
    }]
});
```

---

## Production Checklist

- [ ] **API Key Security**: Never expose API keys in public repositories
- [ ] **Device ID Sync**: Verify Segment `anonymousId` matches G&S device ID
- [ ] **GDPR Compliance**: Only initialize after user consent
- [ ] **Event Forwarding**: Both directions configured (Segment ↔ G&S)
- [ ] **Error Handling**: Graceful fallbacks if SDK fails to load
- [ ] **Testing**: Verify guides/surveys appear for target audience
- [ ] **Debug Mode**: Disable debug logging in production

---

## Complete Implementation Example

```html
<!-- SDK Scripts -->
<script src="https://cdn.amplitude.com/script/YOUR_API_KEY.engagement.js"></script>

<script>
const AMPLITUDE_API_KEY = 'YOUR_API_KEY';

async function initializeGuidesAndSurveys() {
    // Wait for Segment to be ready
    analytics.ready(async function() {
        await new Promise(resolve => setTimeout(resolve, 1000));
        
        // Get Device ID from Segment
        let deviceId = analytics.user?.().anonymousId() || 
                       localStorage.getItem('ajs_anonymous_id')?.replace(/"/g, '');
        
        if (!deviceId) {
            console.error('No device ID available');
            return;
        }
        
        // Boot G&S
        if (window.engagement) {
            await window.engagement.boot({
                user: { device_id: deviceId },
                integrations: [{
                    track: (event) => analytics.track(event.event_type, event.event_properties)
                }]
            });
            
            // Set up event forwarding: Segment → G&S
            analytics.on('track', (event, props) => {
                window.engagement.forwardEvent({ event_type: event, event_properties: props });
            });
            
            analytics.on('page', (category, name, props) => {
                window.engagement.forwardEvent({
                    event_type: '[Amplitude] Page Viewed',
                    event_properties: { ...props, page_name: name }
                });
            });
            
            console.log('✅ Guides & Surveys ready!');
        }
    });
}

// Initialize after consent
document.getElementById('accept-cookies').addEventListener('click', () => {
    analytics.load('YOUR_SEGMENT_WRITE_KEY');
    initializeGuidesAndSurveys();
});
</script>
```

---

## References

- [Amplitude Guides & Surveys SDK Documentation](https://amplitude.com/docs/guides-and-surveys/sdk)
- [Event Forwarding Documentation](https://amplitude.com/docs/guides-and-surveys/sdk#forwarding-events)
- [Segment Analytics.js Documentation](https://segment.com/docs/connections/sources/catalog/libraries/website/javascript/)

---

*Last Updated: January 2026*
*Author: Giuliano Giannini*

