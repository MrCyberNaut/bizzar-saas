# Camera & Video Constraints for WebAR

**Date:** 2026-05-17
**Relevance:** MindAR image quality, detection reliability, camera init robustness

---

## 1. Optimal getUserMedia Constraints for AR

```javascript
const constraints = {
  video: {
    facingMode: { ideal: 'environment' },  // back camera, non-strict
    width:  { ideal: 1280 },
    height: { ideal: 720 },
    frameRate: { ideal: 30, max: 30 },    // cap at 30fps — AR doesn't need 60fps
  },
  audio: false,
}

// With advanced constraints (Chrome Android only, ignore on iOS)
const advancedConstraints = {
  video: {
    facingMode: { ideal: 'environment' },
    width:  { ideal: 1280 },
    height: { ideal: 720 },
    frameRate: { ideal: 30, max: 30 },
    advanced: [
      { focusMode: 'continuous' },       // auto-focus for close-up card scanning
      { exposureMode: 'continuous' },    // stable exposure
      { whiteBalanceMode: 'continuous'}, // stable white balance
    ],
  },
  audio: false,
}

async function initCamera() {
  try {
    return await navigator.mediaDevices.getUserMedia(advancedConstraints)
  } catch (e) {
    // OverconstrainedError — advanced constraints not supported
    return await navigator.mediaDevices.getUserMedia(constraints)
  }
}
```

---

## 2. facingMode — Back Camera Reliability

```javascript
// ideal (not exact) — falls back gracefully if environment not available
facingMode: { ideal: 'environment' }

// exact — throws OverconstrainedError on devices with only one camera
// DON'T USE: facingMode: { exact: 'environment' }
```

**Device quirks:**
- Some Android tablets have no environment camera → fallback to user camera
- iPad in landscape: environment camera maps to unexpected orientation
- Always use `ideal` not `exact`

---

## 3. Resolution Tradeoffs for MindAR

| Resolution | Feature Detection | Performance | Recommendation |
|---|---|---|---|
| 640×480 | Low (fewer features) | Fast | No — too few features |
| 1280×720 | Good | Moderate | ✅ Sweet spot |
| 1920×1080 | Best | Slow on mid-range | No — wasted on small card |

MindAR downsamples internally anyway — 720p is the effective ceiling for feature extraction quality.

---

## 4. focusMode: continuous

Prevents blurry card detection when user moves close:

```javascript
// Check support before applying
const track = stream.getVideoTracks()[0]
const capabilities = track.getCapabilities?.()

if (capabilities?.focusMode?.includes('continuous')) {
  await track.applyConstraints({
    advanced: [{ focusMode: 'continuous' }]
  })
}
```

**Support:**
- Chrome Android: ✅ (most devices)
- Safari iOS: ❌ (iOS manages focus automatically, no API)
- Firefox: ❌

---

## 5. Torch / Flashlight for Low-Light AR

```javascript
const track = stream.getVideoTracks()[0]
const capabilities = track.getCapabilities?.()

if (capabilities?.torch) {
  // Enable torch
  await track.applyConstraints({ advanced: [{ torch: true }] })
  
  // Disable torch
  await track.applyConstraints({ advanced: [{ torch: false }] })
}
```

**Note:** Only Chrome Android. Gate behind capabilities check. iOS: no API.

---

## 6. Camera Permission UX

### Request Pattern
```typescript
// ✅ Request after user interaction — not on page load
function ARStartButton() {
  const handleStart = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: true })
      stream.getTracks().forEach(t => t.stop()) // just checking permission
      // Now initialize full AR
      setARStarted(true)
    } catch (e) {
      if (e.name === 'NotAllowedError') {
        setError('Camera access denied. Please allow camera in browser settings.')
      }
    }
  }
  return <button onClick={handleStart}>Start AR Experience</button>
}
```

### Permission State Check
```typescript
// Check current permission state without prompting
const permissionStatus = await navigator.permissions.query({ name: 'camera' as PermissionName })
// 'granted' | 'denied' | 'prompt'

permissionStatus.onchange = () => {
  if (permissionStatus.state === 'denied') {
    showSettingsRedirectUI()
  }
}
```

### Denied State UX
```typescript
function showSettingsRedirectUI() {
  // iOS: can't programmatically open settings — show manual instructions
  // Android Chrome: navigator.permissions doesn't support programmatic redirect
  alert('Camera access is blocked. Go to Settings → [Browser] → Camera → Allow')
}
```

---

## 7. iOS Safari Quirks

- **Permission not persisted across PWA launches** — re-prompt every session if installed as PWA
- **WKWebView:** Camera blocked entirely (in-app browsers, Telegram, Instagram) → show "open in Safari" prompt
- **No `focusMode` API** — iOS manages focus automatically
- **No `torch` API**
- **getUserMedia works** in Safari 11+ when called from HTTPS

```typescript
// Detect in-app browser (Instagram, TikTok, WeChat, Telegram)
function isInAppBrowser(): boolean {
  const ua = navigator.userAgent
  return /Instagram|FBAV|Twitter|Line|KAKAOTALK|Snapchat/.test(ua)
}

// Show "Open in Safari" banner if in-app browser
if (isInAppBrowser()) {
  showOpenInSafariBanner()
}
```

---

## 8. Android Chrome Quirks

- **Chrome 110+:** Stricter permission model — `getUserMedia` must be called from user gesture
- **Chrome flags:** `#enable-experimental-web-platform-features` exposes advanced constraints
- Standard `getUserMedia` works reliably on Android Chrome 80+
- `NotAllowedError`: permission denied; `NotFoundError`: no camera; `NotReadableError`: camera in use by another app

---

## 9. Stable Tracking via Camera Settings

For MindAR to track consistently, the camera image needs:
- **Stable exposure** — `exposureMode: 'continuous'` prevents flicker from auto-exposure hunting
- **Stable white balance** — `whiteBalanceMode: 'continuous'` prevents color shift
- **In-focus** — `focusMode: 'continuous'` (Android Chrome)

These advanced constraints are all Chrome Android-only. On iOS, the camera hardware manages these automatically — trust the system.

---

## Actionable for BizzAR

1. **Now:** Add `focusMode: 'continuous'` + `exposureMode: 'continuous'` via `applyConstraints` after stream init (Chrome Android) — improves tracking quality in variable lighting
2. **Now:** Add in-app browser detection + "Open in Safari/Chrome" banner — critical for card sharing via Instagram DMs
3. **Now:** Add `frameRate: { max: 30 }` constraint — caps frame rate, saves ~25% battery
4. **v1:** Add camera permission check before AR init — show instructions instead of blank error
5. **v1:** Consider a "low light mode" button that enables torch (Android Chrome) for dark restaurant/bar scenarios

---

## Sources
- https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia
- https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackCapabilities
- https://webkit.org/blog/11555/releasing-the-safari-15-1-technology-preview/ (iOS camera API notes)
- https://chromestatus.com/feature/5642869645541376 (advanced camera constraints)
- https://web.dev/media-capture-and-streams/
