# 01 — MindAR.js Tracking Parameters

**Date:** 2026-05-19
**Topic:** Public tracking configuration knobs in MindAR.js — what they do, defaults, tradeoffs.

---

## TL;DR

MindAR exposes four primary stability knobs plus a few UI/control flags. The smoothing knobs (`filterMinCF`, `filterBeta`) feed directly into a One Euro Filter on the pose; the tolerance knobs (`warmupTolerance`, `missTolerance`) gate detection and loss events. Defaults are tuned for "default WebAR demo" conditions and are **almost always suboptimal for a 3D-content-heavy AR card** like BizzAR.

---

## The Four Smoothing/Tolerance Parameters

### `filterMinCF` — minimum cutoff frequency
- **Default:** `0.001`
- **Role:** Lower bound for the One Euro Filter's adaptive cutoff. Controls how strongly the pose is smoothed when the target is **stationary** (low velocity).
- **Tradeoff:** Lower = less jitter, but more *lag*. Higher = snappier follow, but more visible jitter on a still target.
- **Heuristic:** For a card placed flat on a desk and read at arm's length, target stationary jitter is usually the bigger UX problem. Stay low (`0.001` – `0.01`), tune `filterBeta` upward for responsiveness.

### `filterBeta` — speed coefficient
- **Default:** `1000`
- **Role:** Scales how aggressively the cutoff frequency rises as velocity increases. Higher beta = filter "lets go" faster when the user moves.
- **Tradeoff:** Higher = less lag during motion, slightly more jitter while moving. Lower = sticky/laggy when the user pans the phone.
- **Heuristic:** Default `1000` is high. If content "smears" or "skates" behind the marker when the phone moves, increase further (e.g. `5000`–`10000`). If content jitters wildly during motion, drop it (`100`–`500`).

> A-Frame example from MindAR docs:
> ```html
> <a-scene mindar-image="filterMinCF: 0.1; filterBeta: 10"></a-scene>
> ```
> Source: [MindAR Tracking Config](https://hiukim.github.io/mind-ar-js-doc/quick-start/tracking-config/)

### `warmupTolerance` — frames before "target found"
- **Default:** `5`
- **Role:** The target must be detected in N **continuous** frames before MindAR fires `targetFound`. Prevents flicker from a single noisy frame.
- **Tradeoff:** Higher = fewer false positives, but the AR pops in later. Lower = faster onset, but content may flash if a frame randomly matches.
- **Heuristic:** For a business-card use case (user is consciously aiming), `warmupTolerance: 1` or `2` gives near-instant feel.

### `missTolerance` — frames before "target lost"
- **Default:** `5`
- **Role:** N continuous frames of non-detection before MindAR fires `targetLost`.
- **Tradeoff:** Higher = content stays on-screen during brief occlusion / motion blur (great UX). Lower = quick dismissal.
- **Heuristic:** Crank to `15`–`30`. The card disappearing for half a second because the user tilted is a worse failure mode than content lingering 200 ms too long.

---

## Other Runtime Parameters Worth Knowing

| Param | Default | Notes |
|---|---|---|
| `maxTrack` | `1` | Max simultaneous targets. Set `maxTrack: 2` if a user could see multiple cards in frame. Higher = more CPU. |
| `uiLoading` | enabled | A-Frame builds-in a loading overlay. Set to `'no'` for a custom UI. |
| `uiScanning` | enabled | Built-in scanning reticle. Disable for custom UX. |
| `uiError` | enabled | Built-in error overlay. |
| `autoStart` | `true` | Whether tracking begins on scene load. |
| `imageTargetSrc` | required | URL to compiled `.mind` file. |

Sources: [MindAR docs](https://hiukim.github.io/mind-ar-js-doc/), [react-three-mind README](https://github.com/tommasoturchi/react-three-mind).

---

## What is NOT Exposed (but is in the source)

The public API only exposes the four smoothing/tolerance knobs above. The internal pipeline (`src/image-target/`, `src/controller.js`) has additional knobs that are *not* surfaced as A-Frame/Three.js component attributes:
- detector confidence thresholds
- tracker patch / template sizes
- estimator (pose solve) iteration counts
- TF.js backend selection

See [08-mindar-source-deep-dive.md](./08-mindar-source-deep-dive.md) for what's reachable via patching the library.

---

## Known Tracking-Stability Limitations

[Issue #556 (Unstable AR content)](https://github.com/hiukim/mind-ar-js/issues/556) is an open user complaint where even after tweaking `filterMinCF`/`filterBeta`, content was still unstable. As of search date, no maintainer fix or accepted workaround was posted on the thread. The community-suggested workarounds:
1. Custom client-side smoothing on top of MindAR's output (compose your own filter)
2. Device orientation sensor fusion (gyroscope) to stabilize during quick motion
3. Hybrid tracking (image + IMU or image + SLAM)

---

## Actionable for BizzAR

1. **Stop using defaults.** The defaults (`filterMinCF: 0.001`, `filterBeta: 1000`, `warmupTolerance: 5`, `missTolerance: 5`) are tuned conservatively. For a flat business card viewed under desk lighting:
   - `filterMinCF: 0.0001` (more smoothing at rest — card is static)
   - `filterBeta: 5000`–`10000` (snap quickly when user pans)
   - `warmupTolerance: 2` (fast pop-in)
   - `missTolerance: 20`–`30` (don't dismiss on a tilt)
2. **Expose these in BizzAR's editor** as an "AR Stability" advanced settings panel, with the four knobs above and sensible defaults per use-case.
3. **A/B test stability**: Have two presets — "Stable / lag" vs "Responsive / micro-jitter" — and let creators choose based on their content type (text-heavy panels → Stable; animated 3D model → Responsive).
4. **If `react-three-mind` doesn't forward these props, fork or patch it** — the values are passed through to MindAR's controller. See [05-react-three-mind-internals.md](./05-react-three-mind-internals.md).
5. **Add a client-side wrapper filter** on top of MindAR's pose output for additional smoothing on a per-panel basis (e.g. panels marked "wobble-sensitive" like long text blocks get extra filtering even if the main pose is responsive).

---

## Sources

- [MindAR Tracking Config docs](https://hiukim.github.io/mind-ar-js-doc/quick-start/tracking-config/)
- [MindAR main docs](https://hiukim.github.io/mind-ar-js-doc/)
- [Issue #556 — Unstable AR Content and Ineffective Tracking Configurations](https://github.com/hiukim/mind-ar-js/issues/556)
- [Issue #455 — Multi-tracking AFrame vs Three.js](https://github.com/hiukim/mind-ar-js/issues/455)
- [Three.js forum: Vibrate model in mind-ar](https://discourse.threejs.org/t/vibrate-model-in-mind-ar/62590)
