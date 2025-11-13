[Amplitude-Session-Replay-Segment-Integration-Guide.md](https://github.com/user-attachments/files/23527992/Amplitude-Session-Replay-Segment-Integration-Guide.md)
# Amplitude Session Replay + Segment Integration
## Complete Developer Implementation Guide

**Document Version:** 2.0  
**Last Updated:** November 13, 2024  
**Author:** Giuliano Giannini  
**For:** Customers using Segment as their analytics source

---

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Implementation Steps](#3-implementation-steps)
4. [Critical Configuration Details](#4-critical-configuration-details)
5. [Testing & Validation](#5-testing--validation)
6. [Common Pitfalls & Solutions](#6-common-pitfalls--solutions)
7. [Troubleshooting](#7-troubleshooting)
8. [GDPR & Cookie Consent](#8-gdpr--cookie-consent)
9. [Production Checklist](#9-production-checklist)
10. [FAQ](#10-faq)

---

## 1. Overview

This guide explains how to integrate **Amplitude Session Replay** when using **Segment** as your analytics source. This integration allows you to:

- 📹 Capture user session recordings with video-like playback
- 📊 Link all Segment-tracked events to session replays
- 🔍 Watch exactly what users did when specific events occurred
- 🌍 Maintain proper user attribution (device ID, country, user properties)
- 🐛 Debug user issues by watching actual user sessions

### Architecture Diagram

```
User Browser
    │
    ├─→ Segment Analytics.js (Event Tracking)
    │       ↓
    │   Segment Cloud
    │       ↓
    │   Amplitude (Actions) Destination
    │       ↓
    │   Amplitude Platform (Events stored)
    │
    └─→ Session Replay SDK (Recording)
            ↓
        Amplitude Session Replay API
            ↓
        Amplitude Platform (Replays stored)
```

### Key Difference from Direct Amplitude Integration

| Aspect | Direct Amplitude | With Segment |
|--------|------------------|--------------|
| SDK Type | Session Replay **Plugin** | Session Replay **Standalone** |
| Device ID Management | Auto-synced | **Must manually sync** |
| Session Replay ID | Auto-added to events | **Must inject via middleware** |
| Setup Complexity | ⭐ Easy | ⭐⭐⭐⭐ Advanced |
| Event Tracking | Amplitude SDK | Segment SDK |

**⚠️ CRITICAL:** When using Segment, you MUST use the **Standalone SDK**, not the Plugin!

---

## 2. Prerequisites

### Required Setup in Segment

- [x] **Segment JavaScript Source** configured and sending events
- [x] **Amplitude (Actions) Destination** added to your Segment workspace
- [x] Amplitude destination configured with your Amplitude API Key
- [x] Events flowing from Segment to Amplitude (verify in Segment Debugger)

### Required Setup in Amplitude

- [x] **Amplitude account** with Session Replay enabled
- [x] **Client-Side API Key** (not Server-Side)
- [x] Session Replay quota available

### Technical Requirements

- Modern browser (Chrome, Firefox, Safari, Edge)
- JavaScript enabled
- localStorage support
- Network access to `cdn.amplitude.com` and `cdn.segment.com`

### Information You'll Need

1. **Segment Write Key** - From Segment → Sources → Your Source → Settings → API Keys
2. **Amplitude API Key** - From Amplitude → Settings → Projects → Your Project

---

## 3. Implementation Steps

### Step 1: Load Segment Analytics Snippet

Add the official Segment snippet to your HTML `<head>` or before closing `</body>`:

```html
<!-- Segment Analytics Snippet -->
<script>
!function(){var analytics=window.analytics=window.analytics||[];
if(!analytics.initialize)if(analytics.invoked)window.console&&console.error&&console.error("Segment snippet included twice.");
else{
  analytics.invoked=!0;
  analytics.methods=["trackSubmit","trackClick","trackLink","trackForm","pageview","identify","reset","group","track","ready","alias","debug","page","screen","once","off","on","addSourceMiddleware","addIntegrationMiddleware","setAnonymousId","addDestinationMiddleware","register"];
  analytics.factory=function(e){return function(){
    if(window.analytics.initialized)return window.analytics[e].apply(window.analytics,arguments);
    var n=Array.prototype.slice.call(arguments);
    n.unshift(e);analytics.push(n);return analytics
  }};
  for(var n=0;n<analytics.methods.length;n++){
    var key=analytics.methods[n];
    analytics[key]=analytics.factory(key)
  }
  analytics.load=function(key,n){
    var t=document.createElement("script");
    t.type="text/javascript";
    t.async=!0;
    t.src="https://cdn.segment.com/analytics.js/v1/"+key+"/analytics.min.js";
    var r=document.getElementsByTagName("script")[0];
    r.parentNode.insertBefore(t,r);
    analytics._loadOptions=n
  };
  analytics._writeKey="YOUR_SEGMENT_WRITE_KEY";
  analytics.SNIPPET_VERSION="5.2.0";
  analytics.load("YOUR_SEGMENT_WRITE_KEY");
  analytics.page(); // Remove this line if using cookie consent
}}();
</script>
```

**⚠️ Important Notes:**
- Replace `YOUR_SEGMENT_WRITE_KEY` with your actual Segment write key
- If implementing GDPR cookie consent, remove `analytics.load()` and `analytics.page()` - you'll call these after consent

---

### Step 2: Load Session Replay Standalone SDK

```html
<!-- Amplitude Session Replay Standalone SDK -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

**⚠️ CRITICAL:** 
- Use the **Standalone SDK** URL above
- Do NOT use `plugin-session-replay-browser` (that's for direct Amplitude)
- Do NOT use `unpkg.com` or `jsdelivr.com` - use official Amplitude CDN

**Verify SDK Loaded:**
```javascript
// In browser console, this should return an object (not undefined)
console.log(window.sessionReplay);
```

---

### Step 3: Initialize Session Replay with Segment's Device ID

This is the **MOST CRITICAL** step. Session Replay MUST use the exact same device ID as Segment.

```javascript
const AMPLITUDE_API_KEY = 'YOUR_AMPLITUDE_API_KEY';
const SEGMENT_WRITE_KEY = 'YOUR_SEGMENT_WRITE_KEY';

// Initialize analytics (call after cookie consent if applicable)
async function initializeAnalytics() {
    try {
        // Load Segment
        analytics.load(SEGMENT_WRITE_KEY);
        
        // Force Segment to create device ID immediately
        analytics.identify();
        
        // Wait for Segment to be ready
        analytics.ready(async function() {
            // Give Segment extra time to create anonymousId
            await new Promise(resolve => setTimeout(resolve, 1000));
            
            // Get device ID from Segment - TRY MULTIPLE TIMES
            let deviceId = null;
            let attempts = 0;
            const maxAttempts = 5;
            
            while (!deviceId && attempts < maxAttempts) {
                // Method 1: From Segment user object
                if (analytics.user) {
                    try {
                        deviceId = analytics.user().anonymousId();
                        if (deviceId) {
                            console.log('✅ Device ID from Segment:', deviceId);
                            break;
                        }
                    } catch (e) {
                        console.warn('Error getting device ID:', e);
                    }
                }
                
                // Method 2: From localStorage
                if (!deviceId) {
                    try {
                        const anonId = localStorage.getItem('ajs_anonymous_id');
                        if (anonId) {
                            deviceId = anonId.replace(/"/g, '');
                            console.log('✅ Device ID from localStorage:', deviceId);
                            break;
                        }
                    } catch (e) {
                        console.warn('localStorage error:', e);
                    }
                }
                
                // Wait and retry
                if (!deviceId) {
                    attempts++;
                    console.warn(`⏳ Waiting for device ID (${attempts}/${maxAttempts})...`);
                    await new Promise(resolve => setTimeout(resolve, 500));
                }
            }
            
            // CRITICAL: Don't initialize if we don't have Segment's device ID
            if (!deviceId) {
                console.error('❌ No device ID from Segment - skipping Session Replay');
                return;
            }
            
            // Get session ID
            const sessionId = Date.now(); // Or your session management logic
            
            // Initialize Session Replay
            await window.sessionReplay.init(AMPLITUDE_API_KEY, {
                deviceId: deviceId,        // MUST match Segment!
                sessionId: sessionId,
                sampleRate: 1,             // 100% for testing, 0.1-0.3 for production
                debugMode: true            // Enable for testing
            }).promise;
            
            console.log('✅ Session Replay initialized');
            
            // Verify Session Replay is generating IDs
            const replayProps = window.sessionReplay.getSessionReplayProperties();
            console.log('Session Replay Properties:', replayProps);
            
            // Continue to Step 4 (middleware)...
        });
        
    } catch (error) {
        console.error('Error initializing analytics:', error);
    }
}
```

---

### Step 4: Add Middleware to Inject Session Replay ID

**This is REQUIRED** - without this, events won't link to replays!

```javascript
// Add INSIDE the analytics.ready() callback, AFTER Session Replay initializes

// Middleware 1: Add Session Replay ID to ALL events
analytics.addSourceMiddleware(({ payload, next }) => {
    if (window.sessionReplay) {
        const sessionReplayProps = window.sessionReplay.getSessionReplayProperties();
        
        // Add to track, page, and identify calls
        if (payload.type() === "track" || 
            payload.type() === "page" || 
            payload.type() === "identify") {
            
            payload.obj.properties = {
                ...payload.obj.properties,
                ...sessionReplayProps  // Adds "[Amplitude] Session Replay ID"
            };
            
            // Optional: Log for debugging
            console.log('Session Replay properties added to:', payload.obj.event || payload.type());
        }
    }
    next(payload);
});

// Middleware 2: Update Session ID when Amplitude creates new session
analytics.addSourceMiddleware(({ payload, next }) => {
    setTimeout(() => {
        const nextSessionId = payload.obj?.integrations?.['Actions Amplitude']?.session_id;
        if (nextSessionId && window.sessionReplay) {
            window.sessionReplay.setSessionId(nextSessionId);
            console.log('Session ID updated:', nextSessionId);
        }
    }, 0);
    next(payload);
});

// Track initial page view
analytics.page('Homepage', {
    title: document.title,
    url: window.location.href
});

console.log('✅ Session Replay + Segment integration complete');
```

---

### Step 5: Track Events Normally

```javascript
// Track events through Segment as usual
analytics.track('Button Clicked', {
    button_name: 'Get Started',
    location: 'hero'
});

// Session Replay properties are automatically added via middleware
// No need to manually add [Amplitude] Session Replay ID!
```

---

## 4. Critical Configuration Details

### 🔴 Device ID Synchronization (MOST IMPORTANT)

**Problem:** If Session Replay and Segment use different device IDs, Amplitude cannot link events to replays.

**Why This Happens:**
- Segment creates an `anonymousId` when user first visits
- Session Replay needs a `deviceId` to initialize
- If Session Replay initializes before Segment creates `anonymousId` → mismatch

**Symptoms of Mismatch:**
- ❌ No country data in replays
- ❌ Events don't appear in replay timeline (only "Replay Captured")
- ❌ Amplitude error: "device ID in analytics event does not match device ID in Session Replay ID"

**Solution Checklist:**
```javascript
// ✅ DO: Wait for Segment to create anonymousId
analytics.ready(async function() {
    await new Promise(resolve => setTimeout(resolve, 1000));
    const deviceId = analytics.user().anonymousId();
});

// ✅ DO: Force Segment to create device ID immediately
analytics.identify();

// ✅ DO: Retry multiple times if device ID not ready
while (!deviceId && attempts < 5) {
    // Try to get device ID
    // Wait 500ms
    // Try again
}

// ❌ DON'T: Generate your own device ID
const deviceId = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'; // NEVER DO THIS!

// ❌ DON'T: Initialize Session Replay with null device ID
await window.sessionReplay.init(API_KEY, { deviceId: null }); // NEVER DO THIS!
```

---

### 🔴 Session Replay ID Property Injection

**Problem:** Without `[Amplitude] Session Replay ID` in events, Amplitude can't link them to replays.

**The Property Format:**
```javascript
{
  "[Amplitude] Session Replay ID": "deviceId/sessionId"
  // Example: "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6/1699922971244"
}
```

**How to Add It:**
```javascript
// Get Session Replay properties
const replayProps = window.sessionReplay.getSessionReplayProperties();

// Returns:
// {
//   "[Amplitude] Session Replay ID": "deviceId/sessionId",
//   "[Amplitude] Session Replay Debug": "{...}"
// }

// Add to your event
analytics.track('Event Name', {
    ...yourProperties,
    ...replayProps  // Spread Session Replay properties
});
```

**Using Middleware (Recommended):**
```javascript
// This automatically adds Session Replay ID to EVERY event
analytics.addSourceMiddleware(({ payload, next }) => {
    if (window.sessionReplay && 
        (payload.type() === "track" || payload.type() === "page")) {
        
        const replayProps = window.sessionReplay.getSessionReplayProperties();
        payload.obj.properties = {
            ...payload.obj.properties,
            ...replayProps
        };
    }
    next(payload);
});
```

---

### 🔴 Initialization Order

**Correct Order:**
```
1. Load Segment snippet
2. Load Session Replay SDK script
3. Call analytics.load() [after cookie consent]
4. Call analytics.identify() [forces device ID creation]
5. Wait for analytics.ready()
6. Wait additional 1 second [ensures device ID exists]
7. Retrieve deviceId from Segment
8. Initialize Session Replay with that deviceId
9. Add middleware for property injection
10. Start tracking events
```

**❌ Wrong Order Will Cause:**
- Device ID mismatch
- Null Session Replay IDs
- Events not linking to replays
- Missing country data

---

## 5. Testing & Validation

### Phase 1: Console Validation

**After accepting cookies, you should see:**

```javascript
✅ Retrieved device ID from Segment user(): a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6
✅ Using Segment device ID for Session Replay: a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6
✅ Session Replay initialized successfully!
   Device ID: a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6
   Session ID: 1763042544255
   Sample Rate: 1
   Debug Mode: true
✅ Session Replay ID is being generated: a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6/1763042544255
Segment Analytics & Amplitude Session Replay initialized successfully
Session Replay properties added to event: page
Session Replay properties added to event: Button Clicked
```

**❌ Red Flags to Watch For:**

```javascript
// BAD - Don't see this!
⚠️ Creating device ID - this should match Segment
❌ Could not retrieve Segment device ID after multiple attempts!

// These indicate device ID mismatch issues
```

---

### Phase 2: Segment Debugger Validation

1. Open **Segment** → Your Source → **Debugger**
2. Trigger an event (click a button on your site)
3. Click on the event in the debugger
4. Expand **Properties**
5. **Verify:** `[Amplitude] Session Replay ID` property exists

**Example Event in Segment:**
```json
{
  "event": "Button Clicked",
  "properties": {
    "button_name": "Get Started",
    "location": "hero",
    "[Amplitude] Session Replay ID": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6/1763042544255"
  },
  "userId": null,
  "anonymousId": "a1b2c3d4-e5f6-7g8h-9i0j-k1l2m3n4o5p6",
  "timestamp": "2024-11-13T14:27:40.000Z"
}
```

**✅ Key Validation:**
- Property name is exactly: `[Amplitude] Session Replay ID` (with square brackets)
- Value is a string in format: `"deviceId/sessionId"`
- Value is NOT `null`
- Device ID portion matches `anonymousId` field

---

### Phase 3: Amplitude Ingestion Monitor

1. Go to **Amplitude** → **Session Replay** → **Ingestion Monitor**
2. Wait 5-10 minutes for data to process
3. **All 4 validation checks must be GREEN:**

   ✅ **Remote Configuration Validation**  
   ✅ **Analytics Event Data Validation**  
   ✅ **Session Replay Ingestion Validation**  
   ✅ **Session Replay Data Connection Validation**

**If any checks are RED, see [Troubleshooting](#7-troubleshooting) section.**

---

### Phase 4: Replay Playback Validation

1. Go to **Amplitude** → **Session Replay**
2. Find a recent replay (within last 10 minutes)
3. **Verify the following:**

   ✅ **Country appears** (e.g., "United Kingdom 🇬🇧")  
   ✅ **Device Type appears** (e.g., "Mac", "Windows", "iPhone")  
   ✅ **Multiple events in timeline** (not just "Replay Captured")  
   ✅ **Can click on event** → replay jumps to that moment  
   ✅ **Replay shows actual page interactions** (clicks, scrolls, navigation)

**Example of Correct Replay:**
```
Session ID: 1388245187670
Country: United Kingdom 🇬🇧
Device: Mac
Replay Length: 23s
Event Total: 18 events

Events Timeline:
├─ 1:23:45 PM - Replay Captured
├─ 1:23:46 PM - Viewed Homepage
├─ 1:23:46 PM - Analytics Initialized
├─ 1:23:48 PM - Log In Clicked
├─ 1:23:48 PM - Explore Product Clicked
├─ 1:23:49 PM - Contact Sales Clicked
├─ 1:23:50 PM - Solutions Nav Item Clicked
├─ 1:23:52 PM - Learn More Clicked
├─ 1:23:54 PM - Get Started Clicked
├─ 1:23:55 PM - Stats Section Viewed
└─ ... more events
```

---

## 6. Common Pitfalls & Solutions

### ❌ Pitfall #1: Device ID Mismatch

**Symptoms:**
- No country data in replays
- Events missing from replay timeline
- Only "Replay Captured" event shows
- Amplitude Ingestion Monitor error: "device ID in analytics event does not match"

**Root Cause:**
```javascript
// WRONG - Generating our own device ID
const deviceId = generateUUID();
await window.sessionReplay.init(API_KEY, { deviceId: deviceId });
```

**Fix:**
```javascript
// CORRECT - Using Segment's device ID
const deviceId = analytics.user().anonymousId();
await window.sessionReplay.init(API_KEY, { deviceId: deviceId });
```

**Validation:**
```javascript
// Both should return the SAME value
console.log('Segment:', analytics.user().anonymousId());
console.log('Session Replay:', window.sessionReplay.getSessionReplayProperties());
// Device IDs must match!
```

---

### ❌ Pitfall #2: Missing Session Replay ID in Events

**Symptoms:**
- Amplitude error: "events don't have a Session Replay ID"
- Replays captured but no events linked

**Root Cause:** Middleware not added or incorrect implementation.

**Fix:**
```javascript
// Add this middleware AFTER Session Replay initializes
analytics.addSourceMiddleware(({ payload, next }) => {
    if (window.sessionReplay) {
        const replayProps = window.sessionReplay.getSessionReplayProperties();
        
        if (payload.type() === "track" || payload.type() === "page") {
            payload.obj.properties = {
                ...payload.obj.properties,
                ...replayProps
            };
        }
    }
    next(payload);
});
```

---

### ❌ Pitfall #3: Session Replay ID is Null

**Symptoms:**
```javascript
"[Amplitude] Session Replay ID": null  // In event properties
```

**Root Causes:**
1. Device ID was `null` when initializing Session Replay
2. `sampleRate` is 0 (default - no sessions captured)
3. Page doesn't have browser focus

**Solutions:**
```javascript
// 1. Verify device ID before init
if (!deviceId) {
    console.error('No device ID - cannot initialize');
    return; // Don't initialize!
}

// 2. Set sampleRate for testing
sampleRate: 1  // 100% capture

// 3. Enable debug mode (ignores focus in some browsers)
debugMode: true
```

---

### ❌ Pitfall #4: Using Wrong SDK

**Symptoms:**
- `window.sessionReplay` is undefined
- "sessionReplay.plugin is not a function" error

**Root Cause:**
```html
<!-- WRONG - This is the PLUGIN for direct Amplitude -->
<script src="https://cdn.amplitude.com/libs/plugin-session-replay-browser-*.js"></script>
```

**Fix:**
```html
<!-- CORRECT - Use STANDALONE SDK for Segment -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

**How to Tell:**
```javascript
// Standalone SDK (correct for Segment)
window.sessionReplay.init()        // ✅ Function exists
window.sessionReplay.plugin()      // ❌ Undefined

// Plugin SDK (wrong for Segment)
window.sessionReplay.plugin()      // ✅ Function exists
window.sessionReplay.init()        // ❌ Undefined
```

---

### ❌ Pitfall #5: Timing - Initializing Too Early

**Symptoms:**
- Device ID is always `null`
- Warning: "Segment has not created anonymousId yet"

**Root Cause:**
```javascript
// WRONG - Initializing immediately (Segment not ready)
analytics.load(WRITE_KEY);
const deviceId = analytics.user().anonymousId(); // Returns null!
```

**Fix:**
```javascript
// CORRECT - Wait for Segment
analytics.load(WRITE_KEY);
analytics.identify(); // Force device ID creation

analytics.ready(async function() {
    // Wait extra time
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // NOW device ID is ready
    const deviceId = analytics.user().anonymousId();
});
```

---

### ❌ Pitfall #6: Missing Cookie Consent Integration

**Symptoms:**
- Analytics loads before user accepts cookies
- GDPR compliance issues

**Fix:**
```javascript
// Only initialize after user consent
document.getElementById('accept-cookies').addEventListener('click', function() {
    setCookie('cookie_consent', 'accepted');
    
    // NOW initialize analytics
    initializeAnalytics();
});
```

---

## 7. Troubleshooting

### Issue: "No replays appearing in Amplitude"

**Debug Steps:**

1. **Check Segment is sending data:**
   - Go to Segment → Debugger
   - Verify events are flowing
   - Check events have `[Amplitude] Session Replay ID` property

2. **Check Amplitude destination:**
   - Go to Segment → Connections → Destinations
   - Verify Amplitude (Actions) is enabled
   - Check that events are arriving in Amplitude → Data

3. **Check Session Replay SDK loaded:**
   ```javascript
   console.log(window.sessionReplay); // Should be an object
   ```

4. **Check device ID consistency:**
   ```javascript
   console.log('Segment:', analytics.user().anonymousId());
   console.log('Session Replay:', window.sessionReplay.getSessionReplayProperties());
   // Device IDs must match!
   ```

5. **Check sample rate:**
   ```javascript
   // Must be > 0 to capture sessions
   sampleRate: 1  // or 0.1, 0.3, etc.
   ```

---

### Issue: "Events appearing in Amplitude but not in replays"

**Root Cause:** `[Amplitude] Session Replay ID` is missing or `null` in events.

**Debug Steps:**

1. **Check event in Amplitude:**
   - Go to Data → Event Stream
   - Click on a recent event
   - Check properties for `[Amplitude] Session Replay ID`
   - If missing → middleware not working
   - If `null` → device ID or focus issue

2. **Check middleware is added:**
   ```javascript
   // Run in console
   console.log(analytics._sourceMiddlewares);
   // Should show your middleware functions
   ```

3. **Check getSessionReplayProperties() returns valid data:**
   ```javascript
   const props = window.sessionReplay.getSessionReplayProperties();
   console.log(props);
   // Should return: { "[Amplitude] Session Replay ID": "deviceId/sessionId" }
   // NOT: {} or { "[Amplitude] Session Replay ID": null }
   ```

---

### Issue: "Country data not appearing in replays"

**Root Cause:** Device ID mismatch between Session Replay and Segment events.

**Fix:**
1. Clear cookies and localStorage
2. Refresh page
3. Accept cookies (if applicable)
4. In console, verify:
   ```javascript
   const segmentDeviceId = analytics.user().anonymousId();
   const replayProps = window.sessionReplay.getSessionReplayProperties();
   const replayDeviceId = replayProps['[Amplitude] Session Replay ID'].split('/')[0];
   
   console.log('Segment Device ID:', segmentDeviceId);
   console.log('Replay Device ID:', replayDeviceId);
   console.log('Match:', segmentDeviceId === replayDeviceId); // Must be true!
   ```

---

### Issue: Console Error "sessionReplay.init is not a function"

**Root Cause:** Wrong SDK loaded (Plugin instead of Standalone).

**Fix:**
```html
<!-- Change from: -->
<script src="https://cdn.amplitude.com/libs/plugin-session-replay-browser-*.js"></script>

<!-- To: -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

---

### Issue: "Session Replay ID is null in all events"

**Debug Checklist:**

```javascript
// Run these in console to diagnose

// 1. Check Session Replay initialized
console.log(window.sessionReplay); // Should be object with init, setSessionId, etc.

// 2. Check device ID was provided
// Look in console logs for:
// "Session Replay initialized with deviceId: [some-id]"
// If you see "deviceId: null" → that's the problem!

// 3. Check sample rate
// Sample rate 0 = no sessions captured
// Must be > 0

// 4. Check page focus (for testing)
// Enable debugMode: true to ignore focus

// 5. Manually check what Session Replay returns
const props = window.sessionReplay.getSessionReplayProperties();
console.log(props);
// If returns {} or null → Session Replay not capturing
```

---

## 8. GDPR & Cookie Consent

### Cookie Consent Implementation

**Best Practice:** Only initialize analytics after user consents.

```html
<!-- Cookie Consent Banner -->
<div id="cookie-banner">
    <p>We use cookies and analytics to improve your experience.</p>
    <button id="accept-cookies">Accept All</button>
    <button id="decline-cookies">Decline</button>
</div>

<script>
    // Cookie helper functions
    const cookie = {
        set: (name, value, days = 365) => {
            const expires = new Date(Date.now() + days * 864e5).toUTCString();
            document.cookie = `${name}=${value}; expires=${expires}; path=/`;
        },
        get: (name) => {
            return document.cookie.split('; ')
                .find(row => row.startsWith(name + '='))
                ?.split('=')[1];
        }
    };
    
    // Check consent on page load
    const consent = cookie.get('cookie_consent');
    
    if (consent === 'accepted') {
        // User already consented - initialize immediately
        initializeAnalytics();
    } else if (!consent) {
        // Show cookie banner
        document.getElementById('cookie-banner').style.display = 'block';
    }
    
    // Handle accept
    document.getElementById('accept-cookies').addEventListener('click', function() {
        cookie.set('cookie_consent', 'accepted');
        document.getElementById('cookie-banner').style.display = 'none';
        initializeAnalytics(); // NOW initialize
    });
    
    // Handle decline
    document.getElementById('decline-cookies').addEventListener('click', function() {
        cookie.set('cookie_consent', 'declined');
        document.getElementById('cookie-banner').style.display = 'none';
        // Don't initialize analytics
    });
    
    async function initializeAnalytics() {
        // Load Segment
        analytics.load(SEGMENT_WRITE_KEY);
        analytics.identify();
        
        // Initialize Session Replay (Steps 3-4)
        analytics.ready(async function() {
            // ... Session Replay initialization code from Step 3
        });
    }
</script>
```

---

## 9. Production Checklist

Before deploying to production, verify:

### Configuration
- [ ] Using **Standalone SDK** (not Plugin)
- [ ] Amplitude API key is correct
- [ ] Segment write key is correct
- [ ] Amplitude (Actions) destination enabled in Segment
- [ ] `sampleRate` set appropriately (0.1 - 0.3 for production)
- [ ] `debugMode: false` (disable for production)

### Functionality
- [ ] Cookie consent implemented (if required by GDPR)
- [ ] Analytics only initialize after consent
- [ ] Device ID retrieved from Segment (not generated)
- [ ] Middleware adds `[Amplitude] Session Replay ID` to all events
- [ ] Console shows successful initialization messages
- [ ] No errors in console

### Validation
- [ ] Tested in Segment Debugger - events have Session Replay ID
- [ ] All 4 Amplitude Ingestion Monitor checks are GREEN
- [ ] Sample replay shows country data
- [ ] Sample replay shows multiple events in timeline
- [ ] Can click event in replay and jump to that moment

### Performance
- [ ] No console errors related to Session Replay
- [ ] Page load time acceptable
- [ ] No blocking of main thread
- [ ] Network requests to Amplitude CDN successful

---

## 10. FAQ

### Q: Why is this more complex than direct Amplitude integration?

**A:** With direct Amplitude, the Session Replay Plugin integrates automatically with the Amplitude SDK. Everything is handled for you:
- Device ID syncing ✅ Automatic
- Session Replay ID injection ✅ Automatic
- Session management ✅ Automatic

With Segment, you're using a different analytics provider, so:
- Device ID syncing ❌ Manual
- Session Replay ID injection ❌ Manual (via middleware)
- Session management ⚠️ Semi-manual

**But the benefit:** You get to keep using Segment as your single source of truth for analytics!

---

### Q: What's the performance impact?

**A:** Amplitude Session Replay is optimized for minimal impact:

- **Async processing:** Doesn't block main UI thread
- **Compression:** Lightweight data transfer
- **DOM diffing:** Only captures changes, not full page repeatedly
- **Typical bandwidth:** 50-200 KB per minute of recording
- **CPU impact:** < 5% on average

**Recommendation:** 
- Start with 10-30% sample rate in production
- Monitor performance in real-user monitoring tools
- Adjust sample rate based on traffic volume

---

### Q: How do I handle single-page applications (SPAs)?

**A:** Session Replay works great with SPAs! Just ensure:

1. **Track page changes manually:**
```javascript
// On route change
analytics.page('Product Page', {
    path: window.location.pathname,
    url: window.location.href
});
```

2. **Session continues across page changes** - Session Replay automatically handles this
3. **Device ID persists** - Segment handles this automatically

---

### Q: Can I capture sessions across multiple domains?

**A:** Session Replay works on a single domain. For multi-domain:

- Use **same Amplitude API key** on all domains
- Ensure **device ID persists** across domains (Segment can handle this with proper configuration)
- Implement same Session Replay code on each domain
- Replays will be separate per domain but linked by device ID

---

### Q: What about privacy and PII masking?

**A:** Session Replay automatically masks sensitive data:

**Auto-masked elements:**
- Input fields with type="password"
- Input fields with autocomplete="cc-number" (credit cards)
- Elements with class="amp-mask" or "amp-unmask"

**Custom masking:**
```html
<!-- Mask specific elements -->
<div class="amp-mask">Sensitive content here</div>

<!-- Force unmask (use carefully) -->
<div class="amp-unmask">Safe content</div>
```

**Block entire sections:**
```html
<!-- Session Replay won't capture anything inside this -->
<div class="amp-block">
    <input type="text" placeholder="SSN">
</div>
```

---

### Q: How do I adjust sample rate for production?

```javascript
await window.sessionReplay.init(AMPLITUDE_API_KEY, {
    deviceId: deviceId,
    sessionId: sessionId,
    sampleRate: 0.1,  // 10% of sessions
    debugMode: false  // Disable for production
}).promise;
```

**Recommendations by traffic volume:**
- < 10,000 sessions/month → 50-100% (0.5 - 1.0)
- 10,000 - 100,000 sessions/month → 20-50% (0.2 - 0.5)
- 100,000 - 1M sessions/month → 10-20% (0.1 - 0.2)
- > 1M sessions/month → 5-10% (0.05 - 0.1)

**Note:** Check your Amplitude contract for replay quota limits.

---

### Q: How long does it take for replays to appear?

**A:** Typical processing time:
- **Replay capture:** Real-time (as user interacts)
- **Upload to Amplitude:** Within 1 minute of session end
- **Processing:** 2-5 minutes
- **Available to view:** 5-10 minutes total

**Tip:** In testing, be patient! If you just finished a session, wait 10 minutes before checking Amplitude.

---

### Q: What happens if Session Replay fails to initialize?

**A:** Your site continues to work normally:

- ✅ Segment still tracks events
- ✅ Events still flow to Amplitude
- ❌ Session replays not captured
- ❌ Events won't have `[Amplitude] Session Replay ID`

**Graceful degradation:** Session Replay is additive - if it fails, your analytics still work!

---

### Q: Can I track custom session attributes?

**A:** Yes! Add custom properties to identify calls:

```javascript
// After user logs in or performs key action
analytics.identify(userId, {
    email: 'user@example.com',
    plan: 'enterprise',
    account_age: 30
});

// These properties will be available in Session Replay filters
```

---

### Q: How do I filter replays in Amplitude?

In Amplitude Session Replay, you can filter by:
- Date range
- Country
- Device type
- User properties (from identify calls)
- Events (e.g., show only sessions where user clicked "Checkout")
- Session length
- Event count

**Example:** "Show me all sessions from UK users who clicked 'Get Started' but didn't complete signup"

---

## 📞 Support Resources

### Official Documentation

- [Session Replay Segment Integration](https://amplitude.com/docs/session-replay/session-replay-integration-with-segment)
- [Session Replay Standalone SDK](https://amplitude.com/docs/session-replay/session-replay-standalone-sdk)
- [Segment Analytics.js](https://segment.com/docs/connections/sources/catalog/libraries/website/javascript/)

### Getting Help

If you encounter issues:

1. **Enable verbose logging** (`debugMode: true`)
2. **Check Amplitude Ingestion Monitor** for specific error messages
3. **Review Segment Debugger** to verify event payloads
4. **Contact Amplitude Support** with:
   - Amplitude API Key
   - Segment Source name
   - Example Session Replay ID (from console logs)
   - Browser console logs
   - Screenshot of Ingestion Monitor errors

### Amplitude Support

- **Email:** support@amplitude.com
- **In-app:** Amplitude → Help icon → Contact Support
- **Documentation:** amplitude.com/docs

---

## ✅ Final Implementation Example

Here's a complete, production-ready example:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My App - Segment + Session Replay</title>
    
    <!-- Segment Snippet -->
    <script>
        !function(){var analytics=window.analytics=window.analytics||[];
        if(!analytics.initialize)if(analytics.invoked)window.console&&console.error&&console.error("Segment snippet included twice.");
        else{
          analytics.invoked=!0;
          analytics.methods=["trackSubmit","trackClick","trackLink","trackForm","pageview","identify","reset","group","track","ready","alias","debug","page","screen","once","off","on","addSourceMiddleware","addIntegrationMiddleware","setAnonymousId","addDestinationMiddleware","register"];
          analytics.factory=function(e){return function(){
            if(window.analytics.initialized)return window.analytics[e].apply(window.analytics,arguments);
            var n=Array.prototype.slice.call(arguments);
            n.unshift(e);analytics.push(n);return analytics
          }};
          for(var n=0;n<analytics.methods.length;n++){
            var key=analytics.methods[n];
            analytics[key]=analytics.factory(key)
          }
          analytics.load=function(key,n){
            var t=document.createElement("script");
            t.type="text/javascript";
            t.async=!0;
            t.src="https://cdn.segment.com/analytics.js/v1/"+key+"/analytics.min.js";
            var r=document.getElementsByTagName("script")[0];
            r.parentNode.insertBefore(t,r);
            analytics._loadOptions=n
          };
          analytics._writeKey="YOUR_SEGMENT_WRITE_KEY";
          analytics.SNIPPET_VERSION="5.2.0";
        }}();
    </script>
    
    <!-- Session Replay Standalone SDK -->
    <script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
</head>
<body>
    <h1>My App</h1>
    <button id="cta-button">Get Started</button>
    
    <script>
        const AMPLITUDE_API_KEY = 'YOUR_AMPLITUDE_API_KEY';
        const SEGMENT_WRITE_KEY = 'YOUR_SEGMENT_WRITE_KEY';
        
        // Cookie consent (simplified - implement your own UI)
        const consent = prompt('Accept cookies? (yes/no)');
        
        if (consent === 'yes') {
            initializeAnalytics();
        }
        
        async function initializeAnalytics() {
            try {
                // Load Segment
                analytics.load(SEGMENT_WRITE_KEY);
                
                // Force device ID creation
                analytics.identify();
                
                // Wait for Segment
                analytics.ready(async function() {
                    // Extra wait for device ID
                    await new Promise(resolve => setTimeout(resolve, 1000));
                    
                    // Get device ID from Segment
                    let deviceId = null;
                    let attempts = 0;
                    
                    while (!deviceId && attempts < 5) {
                        if (analytics.user) {
                            deviceId = analytics.user().anonymousId();
                        }
                        
                        if (!deviceId) {
                            const anonId = localStorage.getItem('ajs_anonymous_id');
                            if (anonId) {
                                deviceId = anonId.replace(/"/g, '');
                            }
                        }
                        
                        if (!deviceId) {
                            attempts++;
                            await new Promise(resolve => setTimeout(resolve, 500));
                        }
                    }
                    
                    if (!deviceId) {
                        console.error('No device ID - skipping Session Replay');
                        return;
                    }
                    
                    console.log('Device ID:', deviceId);
                    
                    // Initialize Session Replay
                    await window.sessionReplay.init(AMPLITUDE_API_KEY, {
                        deviceId: deviceId,
                        sessionId: Date.now(),
                        sampleRate: 0.3,  // 30% of sessions
                        debugMode: false
                    }).promise;
                    
                    console.log('Session Replay initialized');
                    
                    // Add middleware for Session Replay ID
                    analytics.addSourceMiddleware(({ payload, next }) => {
                        if (window.sessionReplay) {
                            const replayProps = window.sessionReplay.getSessionReplayProperties();
                            
                            if (payload.type() === "track" || payload.type() === "page") {
                                payload.obj.properties = {
                                    ...payload.obj.properties,
                                    ...replayProps
                                };
                            }
                        }
                        next(payload);
                    });
                    
                    // Track page view
                    analytics.page('Homepage');
                    
                    console.log('Analytics initialized successfully');
                });
                
            } catch (error) {
                console.error('Initialization error:', error);
            }
        }
        
        // Track button click
        document.getElementById('cta-button').addEventListener('click', function() {
            analytics.track('CTA Clicked', {
                button_text: 'Get Started',
                location: 'homepage'
            });
        });
    </script>
</body>
</html>
```

---

## 🎯 Summary - Critical Success Factors

**For Session Replay to work properly with Segment:**

### 1️⃣ Use Standalone SDK (Not Plugin)
```html
✅ https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz
❌ https://cdn.amplitude.com/libs/plugin-session-replay-browser-*.js
```

### 2️⃣ Sync Device IDs Perfectly
```javascript
✅ deviceId = analytics.user().anonymousId() // From Segment
❌ deviceId = generateUUID()                 // Your own
```

### 3️⃣ Wait for Segment Before Initializing
```javascript
✅ analytics.ready(async function() {
     await new Promise(resolve => setTimeout(resolve, 1000));
     // Get device ID here
   })
❌ const deviceId = analytics.user().anonymousId(); // Too early!
```

### 4️⃣ Inject Session Replay ID via Middleware
```javascript
✅ analytics.addSourceMiddleware(({ payload, next }) => {
     const replayProps = window.sessionReplay.getSessionReplayProperties();
     payload.obj.properties = {...payload.obj.properties, ...replayProps};
     next(payload);
   })
❌ // Not adding middleware = events won't link!
```

### 5️⃣ Validate Everything
```javascript
✅ Segment Debugger → Events have [Amplitude] Session Replay ID
✅ Amplitude Ingestion Monitor → All checks GREEN
✅ Amplitude Session Replay → Country + Events appear
❌ Skipping validation = issues in production
```

---

## 🚀 Success Metrics

**When implementation is correct:**

- ✅ Country data appears in every replay
- ✅ 10-50+ events linked to each replay
- ✅ Can click any event → jumps to that moment in video
- ✅ Device ID in replay = Device ID in Segment events
- ✅ All Amplitude Ingestion Monitor checks GREEN
- ✅ Replays available within 10 minutes
- ✅ No console errors

**When implementation has issues:**

- ❌ Only "Replay Captured" event (no other events)
- ❌ Missing country data
- ❌ Device ID mismatch errors in Amplitude
- ❌ `[Amplitude] Session Replay ID` is `null` in events
- ❌ Console warnings about device IDs

---

## 📝 Change Log

**Version 2.0 (November 13, 2024):**
- Added critical device ID synchronization section
- Expanded troubleshooting with specific console commands
- Added GDPR cookie consent implementation
- Included complete working code example
- Added device ID validation procedures
- Expanded FAQ with real-world scenarios

**Version 1.0 (November 13, 2024):**
- Initial release

---

**Document prepared by:** Amplitude Solutions Team  
**For questions or support:** support@amplitude.com

---

## 📎 Appendix A: Quick Reference

### Essential Code Snippet

```javascript
// After cookie consent
analytics.load(SEGMENT_KEY);
analytics.identify(); // Force device ID creation

analytics.ready(async function() {
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    let deviceId = analytics.user().anonymousId() || 
                   localStorage.getItem('ajs_anonymous_id')?.replace(/"/g, '');
    
    if (!deviceId) return console.error('No device ID');
    
    await window.sessionReplay.init(AMPLITUDE_KEY, {
        deviceId: deviceId,
        sessionId: Date.now(),
        sampleRate: 1
    }).promise;
    
    analytics.addSourceMiddleware(({ payload, next }) => {
        if (window.sessionReplay && payload.type() === "track") {
            payload.obj.properties = {
                ...payload.obj.properties,
                ...window.sessionReplay.getSessionReplayProperties()
            };
        }
        next(payload);
    });
});
```

### Console Validation Commands

```javascript
// Check Segment device ID
analytics.user().anonymousId()

// Check Session Replay properties
window.sessionReplay.getSessionReplayProperties()

// Check they match
const segId = analytics.user().anonymousId();
const srId = window.sessionReplay.getSessionReplayProperties()['[Amplitude] Session Replay ID'].split('/')[0];
console.log('Match:', segId === srId); // Must be true!
```

---

**END OF DOCUMENT**

