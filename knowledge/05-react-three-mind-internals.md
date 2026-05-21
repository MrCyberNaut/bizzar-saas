# 05 — react-three-mind Internals

**Date:** 2026-05-19
**Topic:** How `react-three-mind` (by Tommaso Turchi) bridges MindAR into React Three Fiber. Architecture, render-loop coupling, known issues, configuration surface.

URL: https://github.com/tommasoturchi/react-three-mind

---

## TL;DR

`react-three-mind` is a thin React-friendly wrapper around MindAR's Three.js variant. It exposes three components: **`<ARView>`** (wraps R3F `<Canvas>`), **`<ARAnchor>`** (a tracked Object3D anchor), **`<ARFaceMesh>`** (for face tracking).

It works, but it's **lightly maintained** (53 stars, 12 forks, latest v0.3.0 May 2025), and the maintainer's TODO list explicitly calls out "Performance issues requiring fixes" and CI build problems. For a production SaaS, expect to fork.

---

## Component Surface

### `<ARView>`
Wraps R3F's `<Canvas>`. Manages:
- The live camera video as the scene background.
- Camera projection matrix sync with the AR engine.
- MindAR initialization (image or face mode).

Accepts standard `<Canvas>` props plus:
- `tracking: 'image' | 'face'`
- `imageTargets` — URL to compiled `.mind` file
- `maxTrack`
- `filterMinCF`, `filterBeta`, `warmupTolerance`, `missTolerance`
- `autoplay` — auto-start tracking on mount
- `flipUserCamera` — for face tracking
- Methods (via ref): `startTracking()`, `stopTracking()`, `switchCamera()`

### `<ARAnchor>`
An `<Object3D>`-like component that follows a tracked target. Children are positioned in target-local space.
- Props: `target` (target index in the `.mind` file), `onAnchorFound`, `onAnchorLost`
- Children are positioned in target-local space (Z up out of the marker, X right, Y up — conventions vary, verify empirically).

### `<ARFaceMesh>`
Face mesh primitive for face tracking. Customizable material.

---

## Render Loop Coupling

MindAR runs its own per-frame loop:
1. Grab the latest video frame.
2. Run detect / track / pose-solve through the TensorFlow.js WebGL backend.
3. Emit updated anchor poses.

R3F also runs its own render loop. `react-three-mind` couples them by:
- Letting R3F drive the `requestAnimationFrame` cycle.
- MindAR's anchor pose is written into the `<ARAnchor>`'s `Object3D.matrix` (or position/quaternion).
- R3F re-renders on the next frame.

**The two loops run at the same RAF**, but MindAR's CV work and R3F's render compete for the same GPU. On lower-end mobile this serializes badly:
- Frame 1: TF.js detect/track → WebGL contention → R3F render queued late → dropped frame.
- Frame 2: lighter workload → render makes it → judder.

There's no explicit `frameloop="demand"` integration; everything runs at full 60 fps which is wasteful for AR (30 fps would be plenty since most phone cameras cap there).

---

## Known Issues

### From the project itself
The README's TODO calls out:
- Performance issues requiring fixes
- CI build problems

### From the upstream MindAR

[mind-ar-js issue #104 — Decouple ThreeJS from MindAR](https://github.com/hiukim/mind-ar-js/issues/104):
- MindAR's Three.js variant **bundles its own copy of Three.js (r132 at the time of filing)**.
- This causes:
  - Two copies of Three.js loaded in `react-three-mind` apps (one from `three`, one from `mindar-image-three.prod.js`).
  - Version skew bugs when `@react-three/fiber` requires a newer Three.js version than MindAR's bundled one.
  - Bloated bundle size (~600 KB of duplicated Three).
- This has been an open architectural issue for years.

### Practical bugs reported in the community
- [Three.js forum: vibrating models in MindAR](https://discourse.threejs.org/t/vibrate-model-in-mind-ar/62590) — symptom matches the "MindAR defaults are too smoothed at rest, too jumpy in motion" pattern.
- [Issue #455](https://github.com/hiukim/mind-ar-js/issues/455) — Multi-tracking works in A-Frame but not in the Three.js wrapper. Suggests the Three.js bridge has feature parity gaps.

---

## Hidden / Underused Configuration

The wrapper exposes the four common knobs but the **underlying MindAR controller has more**. To reach them you can:
1. Get a ref to the `<ARView>`, dig into its internals to grab the MindAR controller instance.
2. Patch the controller methods (e.g. detection thresholds — see [08-mindar-source-deep-dive.md](./08-mindar-source-deep-dive.md)).
3. Or, fork react-three-mind and add explicit props for whatever else you need.

The wrapper also doesn't expose:
- Camera constraint customization (gets default `getUserMedia`).
- Tensorflow.js backend selection (locked to WebGL).
- Custom `.mind` runtime modification (you need to pre-compile).

---

## Bundle Size / Dependency Hygiene

- **Peer-dep declared:** `@react-three/fiber`, `three`, `react`, `react-dom`.
- **Transitive:** pulls `mindar-image-three` from the MindAR npm publish, which has its own bundled Three (see issue #104).
- A naive `next build` will ship ~1.2 MB of JS for an MindAR + R3F page if no code splitting is done.

---

## Actionable for BizzAR

1. **Lazy-load the AR viewer route.** Use Next.js dynamic import with `ssr: false`:
   ```tsx
   const ARViewer = dynamic(() => import('@/components/ar-viewer'), { ssr: false });
   ```
   Keeps the editor / dashboard bundle clean.
2. **Audit the bundle for duplicate Three.js.** Use `webpack-bundle-analyzer` (or Turbopack equivalent). If MindAR is shipping its own Three, alias-rewrite the import in `next.config.js` to dedupe:
   ```js
   config.resolve.alias['three$'] = require.resolve('three');
   ```
3. **Fork `react-three-mind`** if you need:
   - Custom `getUserMedia` constraints (e.g. force 1280×720 @ 30 fps).
   - Access to the underlying MindAR controller for advanced tuning.
   - Custom filter wrapping on top of MindAR's output.
   - Bug fixes (the maintainer is slow).
4. **Don't use `<ARView>`'s autoplay.** Manually call `startTracking()` after the user taps a "Scan card" CTA — gives a clean permission UX and lets you pre-warm WASM in the background.
5. **Drive AR at 30 fps not 60.** In your R3F `<Canvas>`, set `frameloop="always"` but throttle MindAR's pose-emit to 30 Hz — saves 50% of mobile battery. (May require forking.)
6. **The proto's existing R3F integration in `bizzar-proto`** likely already wires this up — lift it directly. (Per project CLAUDE.md: "Do not duplicate AR logic — lift from proto, don't reinvent.")
7. **Open-source contribution opportunity:** PRs to `react-three-mind` for (a) forwarding all MindAR controller options, (b) deduping Three.js, (c) supporting frameloop="demand" would be welcomed by the small ecosystem.

---

## Sources

- [react-three-mind GitHub](https://github.com/tommasoturchi/react-three-mind)
- [react-three-mind README on UNPKG](https://unpkg.com/browse/react-three-mind@0.2.1/README.md)
- [mind-ar-js issue #104 — Decouple ThreeJS](https://github.com/hiukim/mind-ar-js/issues/104)
- [mind-ar-js issue #455 — Multi-tracking three.js](https://github.com/hiukim/mind-ar-js/issues/455)
- [Three.js forum: Vibrate model in mind-ar](https://discourse.threejs.org/t/vibrate-model-in-mind-ar/62590)
- [R3F performance pitfalls](https://r3f.docs.pmnd.rs/advanced/pitfalls)
- [R3F frameloop revamp proposal](https://github.com/pmndrs/react-three-fiber/issues/2688)
- [R3F: Taking over the render-loop](https://github.com/pmndrs/react-three-fiber/discussions/1339)
