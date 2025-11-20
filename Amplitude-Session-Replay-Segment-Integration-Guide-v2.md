# Amplitude Session Replay + Segment Integration Guide (v2.0)
## Simplified Implementation - No Manual Event Tagging Required

**Author:** Giuliano Giannini  
**Last Updated:** November 20, 2025  
**Implementation:** This Demo Website: https://ggfirenze.github.io/sessionreplaywithsegment/pleo-demo-session-id-only.html

---

## 🎉 What's New in v2.0?

The Session Replay team has simplified the integration! **You no longer need to manually tag every event with `[Amplitude] Session Replay ID`**. 

### Old Approach (Deprecated)
```javascript
// ❌ OLD: Manual middleware to tag EVERY event
segmentAnalytics.addSourceMiddleware(({ payload, next }) => {
    const sessionReplayProperties = sessionReplayInstance.getSessionReplayProperties();
    payload.obj.properties = {
        ...payload.obj.properties,
        ...sessionReplayProperties  // Added [Amplitude] Session Replay ID
    };
    next(payload);
});
```

### New Approach (Current)
```javascript
// ✅ NEW: Automatic linking via Device ID + Time + Session ID
// No manual event tagging middleware required!
// Just ensure Device ID matches and sync Session IDs
```

---

## 🔗 How Automatic Linking Works

Session Replay automatically links to Amplitude events using three mechanisms:

1. **Device ID** (Primary) - Must match between Segment and Session Replay
2. **Time-based linking** - Events link to replays by client timestamp overlap
3. **Session ID** (Safeguard) - Double-check to ensure correct replay association

### Linking Outcomes (from Engineering Documentation)

```
✅ IDEAL: Replay fully linked to events; cohorts & user property search supported
   - Browser captures replay data (in-browser)
   - Browser collects Amplitude Device ID
   - Backend sends Amplitude events
   - Backend includes same Session ID as browser

⚠️ FUNCTIONAL: Replay joinable by time; cohorts & property search possible if user context exists
   - Backend includes Device ID/User ID + Time overlap
   - Some features may be limited

❌ ISOLATED: Replay cannot be connected to analytics
   - Device ID mismatch or no overlap
```

**Our implementation achieves ✅ IDEAL outcome.**

---

## 🚀 Complete Implementation

### 1. Load SDKs

```html
<!-- Segment Analytics Snippet -->
<script>
    !function(){var i="analytics",analytics=window[i]=window[i]||[];
    // ... [Segment snippet code] ...
    analytics._writeKey="YOUR_SEGMENT_WRITE_KEY";
    }}();
</script>

<!-- Amplitude Session Replay Standalone SDK -->
<script src="https://cdn.amplitude.com/libs/session-replay-browser-1.20.1-min.js.gz"></script>
```

### 2. Configuration

```javascript
const SEGMENT_WRITE_KEY = 'YOUR_SEGMENT_WRITE_KEY';
const AMPLITUDE_API_KEY = 'YOUR_AMPLITUDE_API_KEY';

let segmentAnalytics = window.analytics;
let sessionReplayInstance = null;
let analyticsInitialized = false;
```

### 3. Initialize Analytics (After Consent)

```javascript
async function initializeAnalytics() {
    if (analyticsInitialized) return;

    try {
        // Load Segment
        segmentAnalytics.load(SEGMENT_WRITE_KEY);
        
        // Force Segment to create Device ID
        segmentAnalytics.identify();

        segmentAnalytics.ready(async function() {
            // Wait for Segment to fully initialize
            await new Promise(resolve => setTimeout(resolve, 1000));
            
            // CRITICAL: Get Device ID from Segment
            let deviceId = await getSegmentDeviceId();
            
            if (!deviceId) {
                console.error('❌ Could not retrieve Segment device ID!');
                console.error('❌ Skipping Session Replay initialization');
                return;
            }
            
            console.log('✅ Using Segment device ID for Session Replay:', deviceId);

            // Initialize Session Replay with matching Device ID
            sessionReplayInstance = window.sessionReplay;
            
            await sessionReplayInstance.init(AMPLITUDE_API_KEY, {
                sessionId: Date.now(),
                deviceId: deviceId,  // MUST match Segment's Device ID
                sampleRate: 1,
                debugMode: true
            }).promise;

            console.log('✅ Session Replay initialized successfully!');
            console.log('📹 Session Replay is now capturing...');
            console.log('🔗 Events will automatically link via Device ID + Time + Session ID');

            // Setup Session ID sync
            setupSessionIdSync();

            analyticsInitialized = true;

            // Track events normally (no special properties needed!)
            segmentAnalytics.page('Homepage');
            segmentAnalytics.track('Analytics Initialized');
        });

    } catch (error) {
        console.error('Error initializing analytics:', error);
    }
}
```

### 4. Critical: Device ID Retrieval

```javascript
async function getSegmentDeviceId() {
    let deviceId = null;
    let attempts = 0;
    const maxAttempts = 5;
    
    while (!deviceId && attempts < maxAttempts) {
        // Method 1: From Segment's user object
        if (segmentAnalytics.user) {
            try {
                deviceId = segmentAnalytics.user().anonymousId();
                if (deviceId) break;
            } catch (e) {
                console.warn('Error getting device ID from Segment user():', e);
            }
        }
        
        // Method 2: From Segment's localStorage
        if (!deviceId) {
            try {
                const segmentAnonId = localStorage.getItem('ajs_anonymous_id');
                if (segmentAnonId) {
                    deviceId = segmentAnonId.replace(/"/g, '');
                    break;
                }
            } catch (e) {
                console.warn('Error accessing Segment localStorage:', e);
            }
        }
        
        // Retry logic
        if (!deviceId) {
            attempts++;
            console.warn(`⏳ Segment device ID not ready (attempt ${attempts}/${maxAttempts})`);
            await new Promise(resolve => setTimeout(resolve, 500));
        }
    }
    
    return deviceId;
}
```

### 5. Session ID Sync (Still Required)

```javascript
function setupSessionIdSync() {
    // Helper to get stored session ID
    const getStoredSessionId = () => {
        return cookie.get("analytics_session_id") || 0;
    };

    // Sync Session ID when Amplitude creates new session
    segmentAnalytics.addSourceMiddleware(({ payload, next }) => {
        const storedSessionId = getStoredSessionId();
        setTimeout(() => {
            const nextSessionId = payload.obj?.integrations?.['Actions Amplitude']?.session_id || 0;
            if (storedSessionId < nextSessionId) {
                cookie.set("analytics_session_id", nextSessionId);
                sessionReplayInstance.setSessionId(nextSessionId);
                console.log('✅ Session ID updated to:', nextSessionId);
            }
        }, 0);
        next(payload);
    });
}
```

### 6. Track Events Normally

```javascript
// That's it! Just track events as you normally would
// No special Session Replay properties needed

segmentAnalytics.track('Button Clicked', {
    button_name: 'Get Started',
    location: 'hero'
});

segmentAnalytics.track('Page Viewed', {
    page_name: 'Products'
});

segmentAnalytics.track('Form Submitted', {
    form_name: 'Contact Sales'
});

// Events automatically link to replays via Device ID + Time + Session ID
```

---

## ✅ Validation Checklist

### 1. Amplitude Ingestion Monitor
All 4 checks should be **green**:
- ✅ Remote Configuration
- ✅ Analytics Event Data
- ✅ Session Replay Ingestion
- ✅ Session Replay Data Connection

### 2. Session Replay Features
When viewing a replay:
- ✅ **Country/Region** data should be visible (e.g., "United Kingdom")
- ✅ **Timeline** should show multiple events (10-50+ events)
- ✅ Clicking an event should jump to that moment in the replay
- ✅ **Event details** should show full properties

### 3. Segment Debugger
Check events in Segment Debugger:
- ✅ Events should flow through to Amplitude destination
- ✅ **No need** for `[Amplitude] Session Replay ID` property anymore
- ✅ Events should have normal properties only

### 4. Console Logs
Look for these success messages:
```
✅ Retrieved device ID from Segment user(): [device-id]
✅ Session Replay initialized successfully!
   Device ID: [device-id]
   Session ID: [timestamp]
📹 Session Replay is now capturing...
🔗 Events will automatically link via Device ID + Time + Session ID
✨ No manual event tagging required!
```

---

## 🚨 Critical Requirements

### ✅ DO THIS

1. **Match Device IDs** - Session Replay MUST use Segment's `anonymousId` as Device ID
2. **Sync Session IDs** - Update Session Replay SDK when Segment creates new session
3. **Use Standalone SDK** - `session-replay-browser-1.20.1-min.js.gz` (not plugin version)
4. **Initialize after Segment** - Wait for Segment to be ready and have Device ID
5. **Retry Device ID retrieval** - Segment may need time to generate `anonymousId`

### ❌ DON'T DO THIS

1. **Don't generate your own Device ID** - Always use Segment's `anonymousId`
2. **Don't use script loader** - Use manual SDK loading for GDPR compliance
3. **Don't skip retries** - If Device ID is null, Session Replay won't link properly
4. **Don't use plugin version** - Standalone SDK is required for Segment integration
5. **Don't manually tag events** - The old middleware approach is deprecated

---

## 🎯 Key Differences: Old vs New

| Aspect | Old Approach (v1) | New Approach (v2) |
|--------|------------------|-------------------|
| **Event Tagging** | Manual middleware required | ✅ Automatic (no middleware) |
| **Event Property** | `[Amplitude] Session Replay ID` on every event | ✅ Not needed |
| **Device ID** | Must match | ✅ Still must match (critical!) |
| **Session ID** | Sync required | ✅ Still sync required |
| **Linking Method** | Session Replay ID property | ✅ Device ID + Time + Session ID |
| **Code Complexity** | High (2 middlewares) | ✅ Low (1 middleware) |
| **Maintenance** | Complex property tracking | ✅ Simple sync only |

---

## 🔧 Troubleshooting

### Replays show "Unknown" country
**Cause:** Device ID mismatch between Segment and Session Replay  
**Fix:** Verify you're using Segment's `anonymousId`, not a generated ID

### Events not appearing in replay timeline
**Cause:** Time overlap issue or Device ID mismatch  
**Fix:** 
1. Verify Device ID matches (check console logs)
2. Ensure events are being sent to Amplitude
3. Check Segment → Amplitude destination is enabled

### Device ID is null after retries
**Cause:** Segment not fully initialized or localStorage blocked  
**Fix:**
1. Increase retry attempts or wait time
2. Check browser console for Segment errors
3. Verify `analytics.identify()` is called after `analytics.load()`

### Session Replay not initializing
**Cause:** SDK not loaded or incorrect order  
**Fix:**
1. Verify Session Replay SDK script is loaded (check Network tab)
2. Ensure `window.sessionReplay` exists before calling `.init()`
3. Initialize Session Replay inside `segmentAnalytics.ready()` callback

---

## 📊 Expected Results

### Before (Old Approach)
```javascript
// Event in Segment Debugger
{
    "event": "Button Clicked",
    "properties": {
        "button_name": "Get Started",
        "[Amplitude] Session Replay ID": "device-id/session-id"  // Had to manually add this
    }
}
```

### After (New Approach)
```javascript
// Event in Segment Debugger
{
    "event": "Button Clicked",
    "properties": {
        "button_name": "Get Started"
        // No Session Replay ID needed!
    }
}
```

**Both approaches result in the same outcome in Amplitude:**
- ✅ Replay shows correct country
- ✅ Events appear in timeline
- ✅ Clicking event jumps to moment in replay
- ✅ Full event properties visible

**The new approach is just simpler to implement and maintain!**

---

## 🎓 Understanding the Magic

### How does Amplitude link events to replays without the property?

1. **Session Replay SDK** sends replay data to Amplitude with:
   - Device ID (from Segment)
   - Session ID (synced with Segment)
   - Timestamps for all captured activity

2. **Amplitude Events** arrive from Segment with:
   - Device ID (Segment's anonymousId)
   - Session ID (from Amplitude Actions integration)
   - Event timestamps

3. **Amplitude Backend** automatically matches:
   ```
   IF (event.device_id == replay.device_id) 
      AND (event.timestamp OVERLAPS replay.time_window)
      AND (event.session_id == replay.session_id)
   THEN link_event_to_replay()
   ```

4. **Result:** Events appear in replay timeline automatically! 🎉

---

## 🚀 Production Checklist

Before going live:

- [ ] Change `sampleRate` from 1 to 0.1-0.3 (10-30% sampling)
- [ ] Set `debugMode` to `false`
- [ ] Remove API keys from client-side code (use environment variables)
- [ ] Test GDPR cookie consent flow
- [ ] Verify cookie expiration (max 13 months for CNIL compliance)
- [ ] Test on multiple browsers (Chrome, Firefox, Safari)
- [ ] Test on mobile devices
- [ ] Monitor Amplitude ingestion for errors
- [ ] Set up alerts for failed Session Replay ingestion
- [ ] Document Session ID sync logic for team

---

## 📚 Related Resources

- **Amplitude Session Replay Docs:** https://docs.amplitude.com/session-replay
- **Segment Analytics.js:** https://segment.com/docs/connections/sources/catalog/libraries/website/javascript/
- **Amplitude Actions (Segment):** https://segment.com/docs/connections/destinations/catalog/actions-amplitude/

---

## 💡 Tips for Success

1. **Always check Device ID first** - This is the #1 cause of linking issues
2. **Monitor console logs** - They tell you everything about what's working/not working
3. **Test in Incognito** - See the full first-time user experience
4. **Use Amplitude Ingestion Monitor** - Real-time feedback on what's working
5. **Keep it simple** - The new approach is intentionally simpler; don't overcomplicate it

---

## ✨ Summary

The new Session Replay integration is **significantly simpler**:

1. Load Segment + Session Replay SDKs
2. Get Device ID from Segment (CRITICAL!)
3. Initialize Session Replay with matching Device ID
4. Sync Session IDs via middleware
5. Track events normally

**That's it!** No manual event tagging, no complex property management, just clean automatic linking.

---

**Questions or issues?** Check console logs first - they provide detailed information about Device ID retrieval, Session Replay initialization, and Session ID syncing.

**Happy tracking!** 📊🎥

