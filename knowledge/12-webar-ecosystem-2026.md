# WebAR Ecosystem 2026 — Full Landscape

**Date:** 2026-05-17
**Relevance:** Library selection, competitive moat, integration strategy for BizzAR

---

## Full Comparison Table

| Library | iOS | Android | SLAM | Image Tracking | R3F Integration | License | Cost |
|---|---|---|---|---|---|---|---|
| **MindAR.js** | ✅ | ✅ | ❌ | ✅ Best | Via react-three-mind | MIT | Free |
| **react-three-mind** | ✅ | ✅ | ❌ | Via MindAR | ✅ Native | MIT | Free |
| **AR.js** | ✅ | ✅ | ❌ | ✅ NFT | Via artcom wrapper | MIT | Free |
| **8th Wall OSS** | ✅ | ✅ | Binary-only | ✅ | Manual | MIT/Binary | Free (non-commercial) |
| **Zappar SDK** | ✅ | ✅ | ❌ | ✅ | `@zappar/zappar-r3f` | MIT wrapper | ZapWorks sub required |
| **`<model-viewer>`** | AR Quick Look | Scene Viewer | Via ARCore | ❌ | ❌ (web component) | Apache 2.0 | Free |
| **Babylon.js XR** | ✅ | ✅ | Via WebXR | Via WebXR | Manual | Apache 2.0 | Free |
| **WebAR.rocks** | ✅ | ✅ | ❌ | Face/hand only | Manual | Proprietary | Paid tiers |
| **WebXR native** | ❌ Safari | ✅ Chrome | ✅ | Intermittent | Via R3F xr mode | Browser API | Free |
| **Niantic Lightship** | ✅ | ✅ | ✅ | ✅ | ❌ | Proprietary | Enterprise |

---

## Library Deep Dives

### MindAR.js
- **Version:** v1.2.5 (Jan 2024) — last release
- **Maintainer:** hiukim (single maintainer, active on issues)
- **Backend:** TensorFlow.js for feature detection, custom WASM for pose estimation
- **Strengths:** Best-in-class free image tracking for web, works on all browsers
- **Weaknesses:** No SLAM, single maintainer risk, Three.js bundled tightly
- **Issue #104:** Three.js decoupling has been open since 2022 — community pain point
- **npm:** `mind-ar` (~2MB gzipped with WASM)

### react-three-mind
- **Version:** v0.3.0 (May 2025)
- **Maintainer:** tommasoturchi (single, lightly active)
- **What it does:** Wraps MindAR controller inside R3F's Canvas + frame loop
- **Known issues:**
  - Not all MindAR controller options forwarded to ARView props
  - Ships its own Three.js copy → potential version conflicts
  - `jotai` as state manager for anchor pose — adds ~5KB
- **Fork opportunity:** BizzAR could maintain a fork with full props forwarding + Three.js peer dep

### AR.js
- **Status:** Largely unmaintained (last significant commit 2022)
- **Still useful for:** Hiro/barcode markers, GPS-based AR
- **NFT tracking:** Lower quality than MindAR for business cards — fewer features extracted
- **R3F integration:** `@artcom/react-three-arjs` (also lightly maintained)
- **Verdict:** Do not use for BizzAR image tracking. Legacy choice only.

### Zappar WebAR SDK
- **R3F package:** `@zappar/zappar-react-three-fiber` (MIT license)
- **Tracking engine:** Proprietary WASM (requires ZapWorks account for target compilation)
- **Business card example:** `zappar-xr/react-three-example-image-tracking-business-card` (GitHub)
- **Strengths:** Polished SDK, R3F integration is mature, has a business card demo
- **Weaknesses:** Commercial dependency, ZapWorks subscription for production, proprietary CV
- **Verdict:** Good reference for architecture patterns, not viable as free-tier alternative to MindAR

### `<model-viewer>` (Google)
- **What it is:** Web Component for showing 3D models with AR
- **iOS AR:** AR Quick Look (USDZ) — opens natively in Safari, no WebXR needed
- **Android AR:** Scene Viewer via intent (opens ARCore app) or WebXR fallback
- **No image tracking** — requires user to tap to place model in world
- **Verdict for BizzAR:** Strong iOS AR fallback. If WebAR detection fails, link to a `<model-viewer>` with AR Quick Look for iOS users.

### Babylon.js XR
- **Status:** Microsoft-backed, actively maintained, enterprise-grade
- **XR support:** Full WebXR implementation, image tracking via WebXR module
- **vs R3F:** Much heavier bundle (~500KB vs ~150KB R3F), imperative API, steeper learning curve
- **Verdict:** Overkill for BizzAR. R3F is the right choice. But Babylon.js XR docs are excellent reference material for WebXR patterns.

### WebAR.rocks
- **Speciality:** Face tracking, hand tracking, eye tracking via neural nets in WebGL
- **NOT image tracking** — wrong tool for business card AR
- **Interesting tech:** Real-time neural inference in WebGL workers

### Niantic Lightship (post-8th Wall)
- **8th Wall integration was Niantic's web story** — now discontinued
- **Lightship ARDK:** Native mobile only (iOS/Android SDK), no web SDK
- **Enterprise pricing:** Not publicly listed
- **Verdict:** Irrelevant for web-first BizzAR v1/v2

---

## Post-8th Wall Power Vacuum

8th Wall's shutdown created a real gap in the "polished commercial WebAR" market:
- Zappar is the closest remaining paid alternative
- MindAR fills the free/open-source space but has quality limitations
- **Opportunity:** BizzAR can position itself as THE developer-friendly WebAR platform for the business card use case — the niche 8th Wall covered is now fragmented

---

## BizzAR Stack Decision (confirmed correct)

```
MindAR.js (image tracking, free, universal)
  → react-three-mind (R3F bridge)
    → @react-three/fiber (render loop)
      → @react-three/drei (utilities)
        → Three.js (GPU rendering)
```

This is the correct stack for 2026. No credible free alternative exists.

---

## Actionable for BizzAR

1. **Consider forking react-three-mind** — it's 500 lines, single maintainer, has known gaps. A BizzAR fork could add: full props forwarding, peer dep Three.js, better TypeScript types
2. **Add `<model-viewer>` as iOS fallback** — if WebAR fails, show AR Quick Look with USDZ model of user's card + contact info
3. **Monitor MindAR issue #104** — Three.js decoupling would let us use Three.js 0.16x+ (currently pinned to 0.150 for MindAR compat)
4. **Study Zappar's R3F business card example** for architecture patterns (it's the most complete AR business card demo publicly available)
5. **Track 8th Wall OSS commercial license** — if it becomes openly commercial-friendly, their image tracking quality reportedly beats MindAR

---

## Sources
- https://github.com/hiukim/mind-ar-js
- https://github.com/tommasoturchi/react-three-mind
- https://github.com/AR-js-org/AR.js
- https://github.com/zappar-xr/zappar-react-three-fiber
- https://github.com/zappar-xr/react-three-example-image-tracking-business-card
- https://modelviewer.dev
- https://www.babylonjs.com/
- https://8thwall.org
- https://github.com/artcom/react-three-arjs
