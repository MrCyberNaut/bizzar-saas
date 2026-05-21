# 08 — MindAR.js Source Code Deep Dive

**Date:** 2026-05-19
**Topic:** What's inside `hiukim/mind-ar-js`: the tracking pipeline, the controller, the TF.js backend, and what configuration is reachable beyond the public API.

URL: https://github.com/hiukim/mind-ar-js

---

## Project Map

From the project README and search-source analysis:

```
mind-ar-js/
├── src/
│   ├── image-target/        ← All AR algorithms live here
│   │   ├── detectorCPU      ← Older / fallback detector
│   │   ├── detectorGPU      ← Older WebGL2 raw detector
│   │   ├── detectorTF       ← Current detector (TensorFlow.js WebGL)
│   │   ├── trackingGPU      ← Older tracker
│   │   ├── trackingTF       ← Current tracker (TensorFlow.js WebGL)
│   │   ├── estimator        ← Pose solver
│   │   ├── matcher          ← Feature matcher
│   │   └── ...
│   ├── controller.js        ← Top-level API + control flow
│   ├── aframe.js            ← A-Frame component wrapper
│   ├── three.js             ← Three.js wrapper (the one react-three-mind uses)
│   └── compiler.js          ← .mind file generator
│
├── examples/
│   ├── image-tracking/
│   │   ├── compile.html     ← Browser-based target compiler
│   │   └── three.html       ← Three.js usage example
│   └── ...
└── dist/
    ├── mindar-image.prod.js
    ├── mindar-image-aframe.prod.js
    └── mindar-image-three.prod.js
```

Sources:
- [hiukim/mind-ar-js README](https://github.com/hiukim/mind-ar-js/blob/master/README.md)
- [Issue #104 — Decouple Three.js](https://github.com/hiukim/mind-ar-js/issues/104)
- [Core API doc](https://hiukim.github.io/mind-ar-js-doc/core-api/)
- [TypeScript port: krsbx/mind-ar-ts](https://github.com/krsbx/mind-ar-ts) — useful for reading typed signatures

---

## The Tracking Pipeline

A simplified per-frame data flow:

```
camera frame (VideoElement → canvas → tensor)
   ↓
detectorTF
   • ORB-like feature extraction
   • multi-scale pyramid
   • produces feature descriptors for this frame
   ↓
matcher
   • compare against pre-compiled .mind features
   • output: match list with confidences
   ↓
estimator
   • RANSAC + PnP / homography
   • output: 6DOF pose of target in camera space
   ↓
trackingTF (when target already known)
   • inter-frame template tracking
   • cheaper than re-detecting
   ↓
OneEuroFilter (per-axis position + rotation)
   • smoothing via filterMinCF, filterBeta
   ↓
output: Matrix4 → A-Frame entity or Three.js Object3D
```

Two modes:
- **Detection mode** (slow path): when no target is being tracked. Runs the full detector + matcher + estimator. Several frames worth of work.
- **Tracking mode** (fast path): when the target is being followed. Uses cheaper trackingTF + estimator, ~5–10× faster.

`warmupTolerance` gates the transition from detection → tracking.
`missTolerance` gates the transition from tracking → detection (i.e. target lost).

---

## Compute Backend: TF.js WebGL

From README quoted in the search results:
> "The core detection and tracking algorithm is written with custom operations in tensorflowjs. They are like shaders program. This library utilizes tensorflowjs for webgl backend. Tensorflowjs has a very solid webgl engine which allows us to write general purpose GPU application (in this case, our AR application)."

Important consequences:
- **Same WebGL context as Three.js (when shared)** — competes for GPU time. (See [06-webar-mobile-performance.md](./06-webar-mobile-performance.md).)
- **No WebGPU path yet** — even though TF.js has a WebGPU backend now, MindAR's custom ops are WebGL-only.
- **No WASM path** — pure CPU TF.js would be ~10× slower.
- **Custom ops are essentially fragment shaders** running compute via render-to-texture.

The computer-vision algorithm is described as borrowed from ARToolKit ([artoolkitx/artoolkit5](https://github.com/artoolkitx/artoolkit5)).

---

## `src/controller.js` — The Real API

> "Controller is the main API interacting with the AR engine. This section is for advanced users who need to implement highly customized workflow, and it's recommended to refer to the source code of the aframe wrapper to see how it's being used."
> — [MindAR Core API docs](https://hiukim.github.io/mind-ar-js-doc/core-api/)

The `Controller` class:
- Takes a config object with detection thresholds, tracking thresholds, filter parameters.
- Manages the video element + canvas.
- Owns the per-frame loop.
- Exposes events: `onTargetFound`, `onTargetLost`, `onUpdate`, `onFrame`.

Things the controller can do that the public A-Frame/Three.js wrappers don't expose:
- Custom detection confidence thresholds.
- Custom tracking patch size / search window.
- Manual frame stepping (drive the loop yourself).
- Direct access to the TF.js context for sharing buffers.
- Custom video source (e.g. canvas instead of camera — useful for offline processing or for piping a WebRTC remote stream in).

**Workflow if you need this:**
1. Instead of importing `mindar-image-three.prod.js`, import the un-bundled `Controller` from `mind-ar-js`'s source (or use the `mind-ar-ts` TS rewrite).
2. Construct your own R3F integration that consumes `Controller`'s frame events directly.
3. Skip the A-Frame/Three.js wrappers entirely.

---

## `src/compiler.js` — The .mind Generator

Runs in-browser via WASM-backed TF.js ops. Takes one or more images, outputs the `.mind` blob.

Internals (inferred from compile.html example and source):
- Build multi-scale image pyramid (5–6 scales).
- Run feature detector on each scale.
- Compute descriptors.
- Serialize features + descriptors + scale metadata into the `.mind` binary.

You can call the compiler from your own code (it's exposed as `Compiler` class) — useful for **runtime target compilation** where a user uploads a card image and you immediately compile it client-side instead of running it through a server.

---

## Forks / Alternative Builds Worth Knowing

| Fork | Notes |
|---|---|
| [krsbx/mind-ar-ts](https://github.com/krsbx/mind-ar-ts) | Full TypeScript rewrite. Easier to read, similar API. |
| [WonderlandEngine/wonderland-mind-ar-tracking](https://github.com/WonderlandEngine/wonderland-mind-ar-tracking) | Integration for the Wonderland Engine. Reference for how to wire MindAR into a different render engine cleanly. |
| [webarkit/mind-ar-js](https://github.com/webarkit/mind-ar-js) | Community fork, sometimes carries fixes ahead of upstream. |

---

## Open Source Contribution Opportunities

Per project CLAUDE.md ("if we implement something novel … flag it for potential open source contribution"):

1. **Decouple Three.js** ([upstream issue #104](https://github.com/hiukim/mind-ar-js/issues/104), open since 2022). Make Three an external peer dependency.
2. **Expose controller internals via the wrappers**: A PR to mind-ar-js (or react-three-mind) exposing detector thresholds, patch sizes, custom video source, custom filter substitution.
3. **WebGPU backend port**: Significant work, but a real moat.
4. **Image-tracking → WebXR-anchor handoff**: Library that uses MindAR to identify+anchor, then hands off to WebXR for SLAM. This is a likely 8th-Wall-replacement gap.
5. **Frameloop="demand"-friendly mode**: R3F integration that supports both 30 fps cap and on-demand re-rendering.

---

## Actionable for BizzAR

1. **For v1, use the published `mind-ar-js` package** (via `react-three-mind`). Don't fork yet.
2. **Plan a v2 controller-direct integration** for when you need:
   - Custom filter substitution (your own One Euro implementation).
   - Custom detection thresholds (per-card stability tuning).
   - Shared TF.js context with other ML features you might add (face/hand tracking for AR avatars, etc.).
3. **Compile `.mind` files client-side at card-creation time** using `Compiler`. Don't compile on the server — the WASM cost is fine in-browser, and it gives instant feedback to the creator.
4. **Store compiled `.mind` blobs in Supabase Storage** alongside the source image. Re-compile when the source image changes.
5. **Patch react-three-mind to use a forked controller** if/when you need detector thresholds. The patch is mechanical.
6. **Consider `mind-ar-ts`** as a reference if you go the controller-direct route — typed API is much more pleasant.

---

## Sources

- [hiukim/mind-ar-js GitHub](https://github.com/hiukim/mind-ar-js)
- [mind-ar-js README](https://github.com/hiukim/mind-ar-js/blob/master/README.md)
- [MindAR Core API documentation](https://hiukim.github.io/mind-ar-js-doc/core-api/)
- [MindAR Events Handling](https://hiukim.github.io/mind-ar-js-doc/face-tracking-examples/events-handling/)
- [Issue #104 — Decouple ThreeJS from MindAR](https://github.com/hiukim/mind-ar-js/issues/104)
- [Issue #455 — Multi-tracking](https://github.com/hiukim/mind-ar-js/issues/455)
- [krsbx/mind-ar-ts (TypeScript port)](https://github.com/krsbx/mind-ar-ts)
- [WonderlandEngine/wonderland-mind-ar-tracking](https://github.com/WonderlandEngine/wonderland-mind-ar-tracking)
- [@wonderlandengine/mind-ar-tracking on npm](https://www.npmjs.com/package/@wonderlandengine/mind-ar-tracking)
- [webarkit/mind-ar-js fork](https://github.com/webarkit/mind-ar-js)
- [Compile HTML example](https://github.com/hiukim/mind-ar-js/blob/master/examples/image-tracking/compile.html)
- [ARToolKit reference](https://github.com/artoolkitx/artoolkit5)
- [Releases page](https://github.com/hiukim/mind-ar-js/releases)
