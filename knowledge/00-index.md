# BizzAR — WebAR Tracking Knowledge Base

**Date compiled:** 2026-05-19
**Author:** Research session (Opus)
**Scope:** WebAR image tracking quality, MindAR.js, tracking stability, and related topics.

---

## Files in this knowledge base

| # | File | Topic |
|---|---|---|
| 01 | [MindAR Parameters](./01-mindar-parameters.md) | `filterMinCF`, `filterBeta`, `warmupTolerance`, `missTolerance` — what they do, defaults, tradeoffs. |
| 02 | [One Euro Filter](./02-one-euro-filter.md) | The pose-smoothing algorithm under MindAR's hood. Math, pseudocode, tuning procedure. |
| 03 | [WebAR SLAM](./03-webar-slam.md) | What SLAM is, why MindAR doesn't have it, alternatives post-8th-Wall-shutdown. |
| 04 | [Tracking Quality](./04-tracking-quality.md) | Target image design, `.mind` compilation, lighting, camera constraints. |
| 05 | [react-three-mind Internals](./05-react-three-mind-internals.md) | How the R3F wrapper works, known issues, configuration surface. |
| 06 | [Mobile Performance](./06-webar-mobile-performance.md) | GPU contention, RAF throttling, WASM threading, battery, camera tuning. |
| 07 | [WebXR Image Tracking](./07-webxr-image-tracking.md) | Browser-native alternative — viability for production 2026. |
| 08 | [MindAR Source Deep Dive](./08-mindar-source-deep-dive.md) | Internal pipeline, controller API, advanced configuration. |

---

## Top 5 highest-impact changes to make RIGHT NOW

These are the actions that move the most measurable tracking-stability needle for the BizzAR card use case, ranked by `impact / effort`.

### 1. Stop using MindAR's default tracking parameters

**The defaults (`filterMinCF: 0.001`, `filterBeta: 1000`, `warmupTolerance: 5`, `missTolerance: 5`) are wrong for a static, flat business card.**

Use these as a v1 baseline:
```js
{
  filterMinCF: 0.0001,    // more smoothing when card is still
  filterBeta:  5000,      // snap quickly during phone motion
  warmupTolerance: 2,     // fast pop-in
  missTolerance: 25       // ~0.8s grace before content vanishes on a tilt
}
```

This is ~30 minutes of work, and it eliminates the two most common UX complaints: "content feels jumpy when I hold the card still" and "content disappears the second I tilt the phone."

If `react-three-mind` doesn't forward all four props, patch it. See [01-mindar-parameters.md](./01-mindar-parameters.md).

---

### 2. Gate card uploads on a "trackability score"

In the BizzAR card editor, when a user uploads or designs a card:
- Run the MindAR `Compiler` client-side (it's WASM, works in-browser).
- Show the feature heatmap to the user before saving.
- Compute a trackability score: `(feature_count) × (1 − Gini_coefficient_of_distribution)`.
- Refuse to save if the score falls below a threshold (e.g. < 100 effective features).
- Surface a 1–5 star quality rating prominently.

This single change prevents the worst tracking-stability complaints from ever shipping — they're caused by bad targets, not bad tracking config. **A user with a low-contrast wordmark-only card will *never* have stable AR**, no matter what filter you use.

See [04-tracking-quality.md](./04-tracking-quality.md) and [08-mindar-source-deep-dive.md](./08-mindar-source-deep-dive.md) for the Compiler API.

---

### 3. Lock camera constraints to 1280×720 @ 30 fps with continuous autofocus

Replace the default `getUserMedia` call:

```js
navigator.mediaDevices.getUserMedia({
  video: {
    facingMode: { ideal: 'environment' },
    width: { ideal: 1280 },
    height: { ideal: 720 },
    frameRate: { ideal: 30, max: 30 },
    advanced: [{ focusMode: 'continuous' }]
  },
  audio: false
});
```

Cuts GPU/battery load ~30%, removes "blurry when I lean in" failure mode. Also cap the AR render loop at 30 fps. See [06-webar-mobile-performance.md](./06-webar-mobile-performance.md).

---

### 4. Add a `visibilitychange` lifecycle and pre-warm MindAR

Two micro-features that together feel like a different product:
- **Pre-warm**: Load MindAR + TF.js + compile-target-fetch on idle (`requestIdleCallback`), before the user taps "Scan." Eliminates the 500–1500 ms init stall on first scan.
- **Visibility teardown**: Listen for `document.visibilitychange`. On hidden, call `stopTracking()` and release the camera. On visible, resume. Halves battery use during interruptions (notifications, app-switches).

Both are <50 lines of code. See [06-webar-mobile-performance.md](./06-webar-mobile-performance.md).

---

### 5. Wrap MindAR pose output in a custom One Euro Filter

MindAR's internal filter has fixed `d_cutoff` and applies to the raw 6DOF pose uniformly. **Wrap the output in a second filter layer with per-content-type tuning**:

- **Text-heavy panels** (contact info, bio): use a more aggressive smoothing (lower min_cutoff, lower beta). Tolerate ~50ms of lag — they're not animated, the lag is invisible.
- **3D models / animated content**: pass through MindAR's pose unchanged (or with a lighter filter). The motion in the content hides micro-jitter.

This is ~50 lines of JS (`OneEuroFilter` class is small). It gives BizzAR a per-panel-type "tuned for content" feel that pure MindAR can't match. See [02-one-euro-filter.md](./02-one-euro-filter.md) for the full algorithm and code.

---

## Strategic context (for the BizzAR roadmap)

### 8th Wall is dead, long live 8th Wall OSS
- 8th Wall's hosted platform **shut down Feb 28, 2026**.
- It transitioned to open source at **8thwall.org** with a binary-only SLAM engine.
- This is the most significant WebAR event of 2026. Worth a serious evaluation as a SLAM backend for BizzAR v2.

### WebXR image tracking is not a viable primary path
- iOS Safari does not support WebXR (no announced timeline).
- Image-tracking module on Chrome Android is unstable/intermittent.
- The future of browser-native AR is **hit-test + anchors + depth (SLAM-style)**, not image tracking.

### The pragmatic 2026 stack
1. **MindAR** for image identification + initial anchor — universal (iOS + Android).
2. **WebXR hit-test + anchors** as an optional overlay on Android Chrome for persistent world-space content after the marker is seen.
3. **8th Wall OSS SLAM** as a future moat opportunity if you want best-in-class on Android.

### Open-source contribution opportunities surfaced

Per the project CLAUDE.md note about flagging OSS opportunities, these came up during research:
1. **`react-three-mind`**: PR to forward all MindAR controller options, dedupe Three.js (issue #104 has been open since 2022).
2. **`mind-ar-js`**: PR to decouple Three.js from the bundled wrappers.
3. **WebAR ladder library**: A clean "MindAR + WebXR fallback ladder" that detects capability and degrades gracefully would fill a real gap post-8th-Wall.

---

## Notes / Uncertainties

- The internal feature-count thresholds in MindAR (what counts as a "good" target) are not officially documented. The "> 300 features = solid" figure is a community heuristic, not an official spec.
- `react-three-mind`'s exact prop-forwarding behavior for all four filter params needs to be verified empirically — the README and code may not match.
- The 8th Wall OSS license terms for the Distributed Engine Binary need legal review before any integration.
- WebXR Image Tracking module's status is in flux; recheck `chromestatus.com/feature/6548327782940672` before any v2 planning.
- Specific frame budget numbers in [06-webar-mobile-performance.md](./06-webar-mobile-performance.md) are illustrative — measure on your target devices before relying on them.

---

## How to use this kit

1. **Implementer working on a tracking-stability ticket**: read [01-mindar-parameters.md](./01-mindar-parameters.md) and the "Top 5" above. Should be enough for 80% of cases.
2. **Designer/PM working on the card editor**: read [04-tracking-quality.md](./04-tracking-quality.md) for what makes a good vs bad target image.
3. **Architect planning v2**: read [03-webar-slam.md](./03-webar-slam.md), [07-webxr-image-tracking.md](./07-webxr-image-tracking.md), [08-mindar-source-deep-dive.md](./08-mindar-source-deep-dive.md).
4. **Anyone debugging "why is it slow on Pixel 6a"**: read [06-webar-mobile-performance.md](./06-webar-mobile-performance.md).
5. **Anyone tempted to write their own filter**: read [02-one-euro-filter.md](./02-one-euro-filter.md) first — it's already done well.

Cross-link to [[CLAUDE]] and [[bizzar-design]] in Obsidian when consuming this.
