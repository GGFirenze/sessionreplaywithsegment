# Amplitude Guides & Surveys with Segment Integration Guide

> **Implementation Guide for Third-Party Analytics Providers**

This guide documents how to integrate Amplitude Guides and Surveys when using Segment as your primary analytics provider (instead of Amplitude Browser SDK directly). This document is a self-reflection on the work delivered on this demo webpage. It is not official documentation. All customers are strongly encouraged to read Amplitude's official documentation here: https://amplitude.com/docs/guides-and-surveys/sdk#example-of-booting-and-forwarding-events-if-using-segment.
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
2. **Initialization depends on installation method**:
   - **Script loader** (`cdn.amplitude.com/script/API_KEY.engagement.js`): Auto-initializes, only `boot()` needed ✅
   - **npm/yarn package** (`@amplitude/engagement-browser`): Requires both `init()` AND `boot()`
3. **Bidirectional Event Flow**:
   - **G&S → Segment**: Survey events forwarded via `integrations` option
   - **Segment → G&S**: Track/Page events forwarded via `forwardEvent()` for triggers

> ✅ **Verified**: When using the script loader method with Segment, only `boot()` is required. The script loader auto-initializes the SDK. See [working Pleo demo implementation](#complete-implementation-example) for proof.

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

> **Important**: The script loader (`cdn.amplitude.com/script/API_KEY.engagement.js`) **auto-initializes** the SDK when loaded. For Segment integration, you only need to call `boot()` - no explicit `init()` required. This has been verified in production with the Pleo demo.

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

#### Option A: Script Loader Method (Recommended)

When using the script loader, the SDK auto-initializes. You only need to call `boot()`:

```javascript
async function initializeGuidesAndSurveys(deviceId) {
    if (!window.engagement) {
        console.error('Guides & Surveys SDK not loaded');
        return;
    }
    
    // Script loader auto-initializes - just call boot()
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

#### Option B: npm/yarn Package Method

If using the npm package (`@amplitude/engagement-browser`), you must call both `init()` and `boot()`:

```javascript
async function initializeGuidesAndSurveys(deviceId) {
    if (!window.engagement) {
        console.error('Guides & Surveys SDK not loaded');
        return;
    }
    
    // Step 1: Initialize (required for npm package method)
    window.engagement.init(AMPLITUDE_API_KEY, {
        serverZone: 'US',  // or 'EU' for EU data center
    });
    
    // Step 2: Boot with Segment's device ID
    await window.engagement.boot({
        user: {
            device_id: deviceId  // MUST match Segment's anonymousId
        },
        integrations: [{
            track: (event) => {
                analytics.track(event.event_type, event.event_properties);
            }
        }]
    });
    
    console.log('Guides & Surveys initialized and booted with device ID:', deviceId);
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
✅ Guides and Surveys SDK script loaded
✅ Guides and Surveys initialized with API key
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

**Cause**: Calling `init()` when using the script loader (which auto-initializes), or calling `init()` multiple times with npm package

**Solution**: 
- **Script loader**: Don't call `init()` at all - just call `boot()`
- **npm package**: Add a guard to prevent double initialization:

```javascript
// For npm package method only - add guard
if (!window._engagementInitialized) {
    window.engagement.init(AMPLITUDE_API_KEY, { serverZone: 'US' });
    window._engagementInitialized = true;
}

await window.engagement.boot({ user: { device_id: deviceId } });
```

> **Note**: When using the script loader (`cdn.amplitude.com/script/API_KEY.engagement.js`), the SDK auto-initializes. Only call `boot()` - no `init()` needed!

### 2. Guides/Surveys Not Appearing

**Possible causes:**
- Device ID mismatch between Segment and G&S
- `boot()` not called
- No active guides/surveys in Amplitude dashboard
- Targeting rules not met
- Script loader not loaded (check network tab)

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
- [ ] **Script Loader**: Use `cdn.amplitude.com/script/API_KEY.engagement.js` (auto-initializes)
- [ ] **Device ID Sync**: Verify Segment `anonymousId` matches G&S device ID
- [ ] **GDPR Compliance**: Only load SDKs after user consent
- [ ] **Event Forwarding**: Both directions configured (Segment ↔ G&S)
- [ ] **Error Handling**: Graceful fallbacks if SDK fails to load
- [ ] **Testing**: Verify guides/surveys appear for target audience
- [ ] **Debug Status**: Run `window.engagement._debugStatus()` to verify setup
- [ ] **Debug Mode**: Disable debug logging in production

---

## Complete Implementation Example

This example uses the **script loader method** (recommended), which auto-initializes:

```html
<!-- SDK Scripts -->
<script src="https://cdn.amplitude.com/script/YOUR_API_KEY.engagement.js"></script>

<script>
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
        
        // Boot G&S (script loader auto-initializes - no init() needed!)
        if (window.engagement) {
            // Boot with Segment's device ID
            await window.engagement.boot({
                user: { device_id: deviceId },
                integrations: [{
                    track: (event) => analytics.track(event.event_type, event.event_properties)
                }]
            });
            
            console.log('✅ Guides and Surveys booted with device ID:', deviceId);
            
            // Set up event forwarding: Segment → G&S (for event-based triggers)
            analytics.on('track', (event, props) => {
                window.engagement.forwardEvent({ event_type: event, event_properties: props });
            });
            
            analytics.on('page', (category, name, props) => {
                window.engagement.forwardEvent({
                    event_type: '[Amplitude] Page Viewed',
                    event_properties: { ...props, page_name: name }
                });
            });
            
            console.log('🎯 Guides & Surveys ready!');
            console.log('   - G&S events → Segment (via integrations)');
            console.log('   - Segment events → G&S (via forwardEvent)');
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

> ✅ **Verified**: This pattern has been tested and confirmed working in the Pleo demo with surveys appearing correctly and bidirectional event flow operational.

---

## References

- [Amplitude Guides & Surveys SDK Documentation](https://amplitude.com/docs/guides-and-surveys/sdk)
- [Event Forwarding Documentation](https://amplitude.com/docs/guides-and-surveys/sdk#forwarding-events)
- [Segment Analytics.js Documentation](https://segment.com/docs/connections/sources/catalog/libraries/website/javascript/)

---

*Last Updated: January 2026*
*Author: Giuliano Giannini*
