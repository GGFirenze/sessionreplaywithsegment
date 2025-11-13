# Amplitude Session Replay with Segment - Quick Setup Guide

**For customers using Segment as their analytics source**

---

## Overview

This integration enables Session Replay when Segment is your analytics provider. Events flow through Segment to Amplitude, and replays are captured and linked to those events.

**Architecture:** User → Segment (tracking) → Amplitude (events) + Session Replay SDK → Amplitude (replays)

---

## Prerequisites

- Segment JavaScript source configured
- Amplitude (Actions) destination enabled in Segment
- Amplitude API key (client-side)
- Session Replay enabled in your Amplitude account

---

## Implementation Steps

### 1. Load Session Replay Standalone SDK

```html
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

**⚠️ Critical:** Use **Standalone SDK** (not Plugin). Plugin is for direct Amplitude integration only.

---

### 2. Initialize After Segment is Ready

```javascript
const AMPLITUDE_API_KEY = 'your-amplitude-api-key';

// Load Segment first
analytics.load('your-segment-write-key');
analytics.identify(); // Force device ID creation

// Wait for Segment to be ready
analytics.ready(async function() {
    // Wait for device ID to be created
    await new Promise(resolve => setTimeout(resolve, 1000));
    
    // Get device ID from Segment (CRITICAL - must match!)
    let deviceId = analytics.user().anonymousId();
    
    // Fallback to localStorage if needed
    if (!deviceId) {
        const anonId = localStorage.getItem('ajs_anonymous_id');
        if (anonId) deviceId = anonId.replace(/"/g, '');
    }
    
    // Don't initialize if no device ID (prevents mismatch)
    if (!deviceId) {
        console.error('No Segment device ID - skipping Session Replay');
        return;
    }
    
    // Initialize Session Replay with Segment's device ID
    await window.sessionReplay.init(AMPLITUDE_API_KEY, {
        deviceId: deviceId,      // Must match Segment!
        sessionId: Date.now(),
        sampleRate: 1            // 100% for testing, 0.1-0.3 for production
    }).promise;
    
    console.log('✅ Session Replay initialized with device ID:', deviceId);
    
    // Continue to Step 3...
});
```

---

### 3. Add Middleware to Inject Session Replay ID

**This step is REQUIRED** - without it, events won't link to replays.

```javascript
// Add this inside analytics.ready(), AFTER Session Replay initializes

analytics.addSourceMiddleware(({ payload, next }) => {
    if (window.sessionReplay) {
        const replayProps = window.sessionReplay.getSessionReplayProperties();
        
        // Add Session Replay ID to track and page events
        if (payload.type() === "track" || payload.type() === "page") {
            payload.obj.properties = {
                ...payload.obj.properties,
                ...replayProps  // Adds "[Amplitude] Session Replay ID"
            };
        }
    }
    next(payload);
});

// Track page view
analytics.page('Homepage');
```

---

## Critical Success Factors

### 🔴 Device ID Must Match

**The #1 cause of Session Replay issues:**

- Segment uses `anonymousId` as device ID
- Session Replay must use the SAME `deviceId`
- If they don't match → no country data, events don't link to replays

**✅ Correct:**
```javascript
const deviceId = analytics.user().anonymousId(); // From Segment
```

**❌ Wrong:**
```javascript
const deviceId = generateUUID(); // Your own - DON'T DO THIS!
```

---

### 🔴 Session Replay ID Must Be in Every Event

**Without this property, events won't appear in replays:**

```javascript
"[Amplitude] Session Replay ID": "deviceId/sessionId"
```

**How to add:** Use middleware (Step 3) to automatically inject into all events.

**Verify:** Check Segment Debugger → Event should have `[Amplitude] Session Replay ID` property.

---

## Validation

### 1. Console Check
```javascript
✅ Session Replay initialized with device ID: abc123-def456-...
✅ Session Replay ID is being generated: abc123-def456-.../1763042544255
Session Replay properties added to event: page
```

### 2. Segment Debugger
- Events show `[Amplitude] Session Replay ID` property
- Value is string "deviceId/sessionId" (not null)

### 3. Amplitude Ingestion Monitor
- All 4 validation checks GREEN
- Go to: Amplitude → Session Replay → Ingestion Monitor

### 4. Replay Playback
- Country appears (e.g., "United Kingdom 🇬🇧")
- Multiple events in timeline (10-50+ events)
- Can click event → jumps to that moment

---

## Common Issues & Quick Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Device ID mismatch** | No country, only "Replay Captured" event | Use `analytics.user().anonymousId()` |
| **Missing Session Replay ID** | Events don't link to replays | Add middleware (Step 3) |
| **Session Replay ID is null** | Property exists but value is null | Ensure device ID not null before init |
| **Wrong SDK** | `window.sessionReplay` undefined | Use Standalone SDK, not Plugin |
| **Timing issue** | Device ID always null | Wait in `analytics.ready()` + `identify()` |

---

## Comparison: Direct Amplitude vs Segment

| Feature | Direct Amplitude | With Segment |
|---------|------------------|--------------|
| SDK | Session Replay **Plugin** | Session Replay **Standalone** |
| Device ID sync | Automatic ✅ | Manual (must sync) ⚠️ |
| Event property injection | Automatic ✅ | Manual (middleware) ⚠️ |
| Complexity | Easy ⭐ | Advanced ⭐⭐⭐⭐ |

---

## Minimal Working Example

```html
<!DOCTYPE html>
<html>
<head>
    <!-- Segment -->
    <script>
        !function(){var analytics=window.analytics=window.analytics||[];
        /* Segment snippet */
        analytics._writeKey="YOUR_SEGMENT_KEY";
        }}();
    </script>
    
    <!-- Session Replay Standalone SDK -->
    <script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
</head>
<body>
    <button onclick="analytics.track('Button Clicked')">Click Me</button>
    
    <script>
        const AMPLITUDE_KEY = 'your-amplitude-key';
        
        analytics.load('YOUR_SEGMENT_KEY');
        analytics.identify();
        
        analytics.ready(async () => {
            await new Promise(r => setTimeout(r, 1000));
            
            const deviceId = analytics.user().anonymousId();
            if (!deviceId) return;
            
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
            
            analytics.page();
        });
    </script>
</body>
</html>
```

---

## Key Takeaways

1. ✅ Use **Standalone SDK** for Segment integration
2. ✅ Device ID from **Segment** must match Session Replay device ID
3. ✅ Call `analytics.identify()` to force device ID creation
4. ✅ Add **middleware** to inject `[Amplitude] Session Replay ID` to all events
5. ✅ Validate in **Amplitude Ingestion Monitor** (all checks green)
6. ✅ Verify replays show **country + events** in timeline

**When done correctly:** Replays capture user sessions with full event timeline and geo data.

---

## Resources

- [Official Documentation](https://amplitude.com/docs/session-replay/session-replay-integration-with-segment)
- **Support:** support@amplitude.com
- **Ingestion Monitor:** Amplitude → Session Replay → Ingestion Monitor

---

**Version:** 2.0 | **Updated:** November 13, 2024

