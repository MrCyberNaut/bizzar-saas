# SLAM for WebAR — Deep Dive

**Date:** 2026-05-17
**Relevance:** BizzAR v2 architecture, tracking stability, "sticky mode" after card scan

---

## 1. SLAM vs Image Tracking — The Algorithmic Difference

### Image Tracking (what MindAR does)
1. Extract local feature descriptors (BRIEF/ORB) from a reference image
2. On each camera frame, find matching features in the live feed
3. Solve PnP (Perspective-n-Point) to estimate 6DOF pose from matched features
4. Apply One Euro Filter to smooth the pose

**Limitations:**
- Requires the target image to be in frame at all times
- No "memory" — if the card is occluded for 1 frame, tracking must restart
- Accuracy degrades with viewing angle, motion blur, and poor lighting
- CPU-bound feature matching on every frame

### SLAM (Simultaneous Localization and Mapping)
1. Build a 3D map of the environment from camera frames
2. Localize the camera within that map frame-to-frame
3. Key components: **front-end** (feature tracking, odometry) + **back-end** (graph optimization, loop closure)
4. Works even when the original target is out of frame

**For BizzAR use case:**
- SLAM could maintain AR panel positions even when user's hand covers the card
- Enables "place and walk away" — panels stay fixed in world space
- Requires significantly more compute than image tracking

### Visual-Inertial Odometry (VIO)
Combines camera + IMU (accelerometer/gyroscope) for stable tracking:
- **Camera** provides accurate but latency-prone pose
- **IMU** provides low-latency but drift-prone orientation
- Fusion via Extended Kalman Filter or factor graph optimization
- Apple ARKit and Google ARCore both use VIO
- **Browser access:** `DeviceMotionEvent` + `DeviceOrientationEvent` (iOS requires permission prompt since iOS 13)

---

## 2. 8th Wall — Shutdown & OSS Transition

### What happened
- 8th Wall was the leading commercial WebAR SLAM platform
- **Shut down Feb 28, 2026** (Niantic acquisition → discontinuation)
- Transitioned to open source at **8thwall.org**

### What's open source (MIT)
- Image Targets tracking
- Face Effects
- World Effects (limited)
- The three.js / A-Frame / Babylon.js integration layers

### What's binary-only (commercial restrictions)
- The SLAM engine ("Distributed Engine Binary")
- VIO-based world tracking
- Persistent anchors
- **License:** Free for non-commercial; commercial use requires separate agreement

### Implications for BizzAR
- Can use 8th Wall's Image Targets as a potential MindAR alternative (MIT)
- SLAM engine requires commercial license negotiation — not viable for v1
- The open-source community support is uncertain post-Niantic shutdown

---

## 3. WebXR Device API — Browser Support 2026

### Image Tracking Module
```
Chrome Android: Available (behind flag, intermittent)
Safari iOS: NOT SUPPORTED (no WebXR at all)
Samsung Internet: Partial
Firefox: No
```

**Verdict: Cannot rely on WebXR image tracking as primary — iOS is 50%+ of users.**

### Hit Test + Anchors (SLAM-lite)
```
Chrome Android 81+: ✅ hit-test, ✅ anchors
Safari iOS: ❌ (zero WebXR support, no announced timeline)
Samsung Internet 11+: ✅ partial
```

### Hybrid Approach (MindAR + WebXR Anchors)
Use MindAR to identify the card → hand off to WebXR anchors for persistent world-space tracking on Android:

```typescript
// After MindAR detects card, create WebXR anchor at that pose
async function createPersistentAnchor(pose: XRPose, frame: XRFrame) {
  if (!frame.createAnchor) return null // iOS fallback
  const anchor = await frame.createAnchor(pose.transform, frame.referenceSpace)
  return anchor
}

// In R3F, check for WebXR support
const { gl } = useThree()
const xrSession = gl.xr.getSession()
const supportsAnchors = xrSession?.enabledFeatures?.includes('anchors')
```

### ARCore (Android) / ARKit (iOS) Web Bridges
- **ARCore:** Exposed via WebXR on Chrome Android — hit-test works well
- **ARKit:** No web bridge. WKWebView blocks WebXR entirely.
- **Reality Composer / AR Quick Look (iOS):** Only works via `<model-viewer>` or direct USDZ link, not programmatic WebXR

---

## 4. Open Source SLAM Alternatives

### AlvaAR
- ORB-SLAM2 compiled to WASM
- License: GPLv3 (incompatible with commercial SaaS — must open source your code or buy commercial license)
- Performance: ~15fps on mid-range phones, heavy battery drain
- GitHub: https://github.com/AlvaAR/alva_ar
- **Verdict:** GPL license is a blocker for BizzAR commercial use

### ORB-SLAM3
- Academic, C++, no WASM port maintained
- Research use only

### OpenVSLAM / stella_vslam
- C++, BSD-2 license
- No production WASM port

### Pragmatic verdict for BizzAR
No viable open-source SLAM runs well in browser at commercial scale in 2026. The options are:
1. MindAR image tracking (current — works, limited)
2. 8th Wall OSS SLAM binary (needs commercial negotiation)
3. Build WebXR-only SLAM path for Android-only "premium" experience

---

## 5. BizzAR Roadmap Recommendations

### v1 (now): MindAR image tracking
- Optimize filter params (done)
- Improve target image quality guidance
- `missTolerance: 25` effectively simulates short-term SLAM persistence

### v2: Hybrid MindAR + WebXR anchors
```
iOS: MindAR only (card must stay in frame)
Android Chrome: MindAR detection → WebXR anchor creation → SLAM persistence
```
- Detection with MindAR (universal)
- After 1s of stable tracking, create WebXR anchor at card pose
- AR panels then persist in world space even if card is covered

### v3+: Evaluate 8th Wall OSS SLAM binary
- Once commercial terms are clear
- Could enable "place card face-down, panels stay in room" UX

---

## Actionable for BizzAR

1. **Now:** Add `DeviceMotionEvent` + `DeviceOrientationEvent` permission request on iOS for future VIO use
2. **Now:** Check `navigator.xr?.isSessionSupported('immersive-ar')` at load time — log for analytics (what % of users have WebXR)
3. **v2:** Implement WebXR anchor creation after stable MindAR detection on Android
4. **v2:** Add `<model-viewer>` fallback for iOS that shows AR Quick Look with the user's card as USDZ
5. **Avoid:** Building on 8th Wall SLAM until commercial license terms are public

---

## Sources
- https://github.com/hiukim/mind-ar-js
- https://8thwall.org (8th Wall OSS transition)
- https://chromestatus.com/feature/6548327782940672 (WebXR Image Tracking)
- https://developer.apple.com/documentation/arkit (ARKit — iOS native only)
- https://github.com/AlvaAR/alva_ar (ORB-SLAM2 WASM)
- https://immersiveweb.dev (WebXR browser compatibility)
- https://developer.chrome.com/docs/web-platform/webxr (Chrome WebXR docs)
