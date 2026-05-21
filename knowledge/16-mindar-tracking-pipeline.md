# MindAR Tracking Pipeline — Deep Research

## Overview

MindAR image tracking is a **purely JavaScript, GPU-accelerated** pipeline — there is no native WASM module. It uses **TensorFlow.js as a WebGL compute engine** (not for machine learning), executing custom GPU shaders for computer vision operations. This is the key architectural insight: TF.js provides the WebGL abstraction, and MindAR writes general-purpose GPU kernels on top of it.

---

## Package Structure

```
mind-ar (npm)
├── dist/
│   ├── mindar-image.prod.js           ← Core image tracking (R3F / vanilla Three.js)
│   ├── mindar-image-aframe.prod.js    ← A-Frame wrapper
│   ├── mindar-face.prod.js            ← Face tracking (MediaPipe)
│   └── mindar-face-aframe.prod.js
├── src/
│   ├── image-target/
│   │   ├── compiler.js                ← Browser-side Compiler class
│   │   ├── compiler-base.js           ← Base: grayscale, pyramid, export/import
│   │   ├── compiler.worker.js         ← Web Worker for off-thread compilation
│   │   ├── controller.js              ← Main runtime controller
│   │   ├── matching/
│   │   │   ├── matcher.js             ← Target matching orchestrator
│   │   │   └── matching.js            ← BRIEF-style descriptor matching
│   │   └── tracking/
│   │       └── tracker.js             ← Frame-to-frame tracking
│   └── face-target/
│       └── ...                        ← MediaPipe face mesh
└── custom_tfjs/                       ← Custom TensorFlow.js ops (GPU shaders)
```

**Version:** 1.2.5 (released January 16, 2024). Written in pure JavaScript (95% JS, 4.1% HTML). No WASM.

---

## The Full Tracking Pipeline

### Phase 1: Compilation (offline, one-time)

Run before deployment (or on card upload in BizzAR).

```
Input Image
    │
    ▼
Grayscale Conversion   (RGB averaging via canvas pixel manipulation)
    │
    ▼
Image Pyramid          (multiple downscaled versions for scale-invariance)
    │
    ▼
Keypoint Detection     (scale-space extrema — local maxima/minima of DoG-like filter)
    │           ├── maximaPoints  ─┐
    │           └── minimaPoints  ─┴──► matchingData (KPM features)
    │
    ▼
Descriptor Computation (BRIEF-style binary patch descriptors)
    │
    ▼
Tracking Point Set     (lightweight, sparse subset for frame tracking)
    │
    ▼
.mind file export      (msgpack binary, version 2)
```

### Phase 2: Runtime — Detection (first lock)

Runs every frame until the target is found.

```
Camera Frame
    │
    ▼
Input Tensor (via TF.js WebGL backend)
    │
    ▼
Crop Detector            (GPU-accelerated feature detection on live frame)
    │
    ▼
Feature Point Extraction (fast keypoints from current frame)
    │
    ▼
Matching (matcher.js)    (compare live features to stored .mind descriptors)
    │
    ▼  ← only runs if numTracked < maxTrack
Homography Estimation    (compute H from matched point pairs)
    │
    ▼
Screen → World Transform (project to 3D world coordinates via H)
    │
    ▼
Warmup Tolerance Check   (must detect successfully for N consecutive frames)
    │
    ▼
Target Found → emit onUpdate
```

### Phase 3: Runtime — Tracking (active lock)

Once detected, switches to a faster tracking path.

```
Camera Frame
    │
    ▼
tracker.js               (tracks known points frame-to-frame)
    │
    ▼
trackUpdate()            (refine modelViewTransform using new frame data)
    │
    ▼
OneEuroFilter            (smooth jitter in position/rotation)
    │
    │   filterMinCF  — controls high-frequency cutoff (reduce jitter: lower value)
    │   filterBeta   — controls response speed (reduce lag: higher value)
    │
    ▼
World Matrix Update      (new 4x4 modelView matrix)
    │
    ▼
onUpdate callback        (provides matrix to Three.js / R3F)
```

### Phase 4: Loss Detection

```
Target Lost for N frames (missTolerance)
    │
    ▼
Return to Detection Phase
```

---

## Feature Extraction — What Algorithm?

MindAR does NOT use ORB or SIFT by name, but the pipeline is functionally equivalent:

| Step | MindAR approach |
|---|---|
| Keypoint detection | Scale-space extrema (similar to DoG used in SIFT) — GPU kernel via TF.js |
| Descriptor | BRIEF-style binary descriptors — patch comparison stored as bit vectors |
| Matching | Brute-force nearest neighbour on binary descriptors (Hamming distance equivalent) |
| Pose estimation | Homography matrix computation from matched point pairs |
| Tracking filter | OneEuroFilter (adaptive frequency low-pass filter) |

The actual GPU kernel code lives in `custom_tfjs/` as custom TF.js operations — not in a separate WASM binary.

---

## The Controller API (Full Surface)

### Constructor

```js
const controller = new Controller({
  inputWidth,           // number — camera feed width
  inputHeight,          // number — camera feed height
  onUpdate,             // (data: UpdateData) => void
  debugMode,            // boolean (default: false)
  maxTrack,             // number (default: 1)
  warmupTolerance,      // number — frames before target shown (default: 5)
  missTolerance,        // number — frames before target hidden (default: 5)
  filterMinCF,          // number — OneEuroFilter cutoff (default: 0.001)
  filterBeta,           // number — OneEuroFilter speed (default: 1000)
});
```

### Public Methods

```js
// Load targets
await controller.addImageTargets(fileURL);
await controller.addImageTargetsFromBuffer(buffer);

// Start/stop
controller.processVideo(videoElement);  // starts the render loop
controller.stopProcessVideo();

// Single-frame operations (advanced)
const featurePoints = await controller.detect(inputTensor);
const matchResult = await controller.match(featurePoints, targetIndex);
await controller.track(input, modelViewTransform, targetIndex);
await controller.trackUpdate(modelViewTransform, trackFeatures);

// Matrix access
const projMatrix = controller.getProjectionMatrix();
const worldMatrix = controller.getWorldMatrix(modelViewTransform, targetIndex);

// Warmup (important for perf)
await controller.dummyRun(inputTensor);  // pre-warms GPU kernels

// Debug
controller.showTFStats();  // TF.js memory usage

// Cleanup
controller.dispose();
```

### Undocumented option

```js
controller.interestedTargetIndex = 2;
// Default: -1 (match against all targets)
// Set to a specific index to only match one target — useful for perf optimization
```

### `onUpdate` callback shape

```js
// Called per frame with the tracking result for ALL targets
function onUpdate({ type, targetIndex, modelViewTransform }) {
  if (type === 'updateMatrix') {
    // type is 'updateMatrix' when target is visible
    const worldMatrix = controller.getWorldMatrix(modelViewTransform, targetIndex);
    // worldMatrix is a flat Float32Array[16] — standard column-major 4x4
    threeJsObject.matrix.fromArray(worldMatrix);
    threeJsObject.matrix.decompose(
      threeJsObject.position,
      threeJsObject.quaternion,
      threeJsObject.scale
    );
  }
}
```

---

## Multi-Target Tracking — How `maxTrack` Works

```
maxTrack = N
│
▼
Each frame: count currently tracked targets
│
├── If count < N → run detection + matching for untracked targets
│
└── If count >= N → skip detection, only update existing tracks
```

This means:
- Every additional tracked target costs roughly the same CPU/GPU budget as the first
- The detection phase (expensive: feature extraction + matching) only runs on non-tracked targets
- Setting `maxTrack = 1` is the most CPU-efficient configuration
- On mid-range mobile: `maxTrack = 2` is comfortable; `maxTrack = 3+` will cause frame drops

For BizzAR (single business card per AR experience): **keep `maxTrack = 1`**.

---

## The `.mind` File Format — Internals

```
[0x02]                         ← version byte (CURRENT_VERSION = 2)
[msgpack-encoded blob]         ← rest of file
  {
    v: 2,
    dataList: [
      {
        targetImage: { width: W, height: H },  // original image dimensions
        trackingData: [                          // per pyramid level
          { points: [{x, y}, ...] }
        ],
        matchingData: [                          // per pyramid level
          {
            maximaPoints: [...],   // keyframes with descriptors
            minimaPoints: [...],
            width, height, scale   // level dimensions and scale factor
          }
        ]
      }
    ]
  }
```

Raw pixel data is **not** stored. Typical file size: 50–250 KB per target.

---

## How `react-three-mind` Wraps the Controller

`react-three-mind` (by tommasoturchi) is a thin React wrapper that:

1. Creates an `ARView` which wraps `@react-three/fiber`'s `Canvas` — it handles the video background and aligns the Three.js camera projection to match the physical camera
2. Exposes an `ARAnchor` component — a `Object3D` (Three.js Group) whose matrix is updated each frame from the `onUpdate` callback
3. Passes config props (`imageTargets`, `maxTrack`, `filterMinCF`, `filterBeta`, `warmupTolerance`, `missTolerance`) through to the underlying controller
4. Exposes imperative handles: `startTracking()`, `stopTracking()`, `switchCamera()`

**Frame matrix update flow:**
```
MindAR onUpdate callback
  → controller.getWorldMatrix()
  → anchor.group.matrix.fromArray()
  → anchor.group.matrix.decompose(position, quaternion, scale)
  → Three.js re-renders with updated transforms
```

The camera projection matrix is set once at start via `controller.getProjectionMatrix()` and synced with `camera.projectionMatrix`.

---

## Known Accuracy Limitations

| Limitation | Detail |
|---|---|
| **Planar target assumption** | The tracking assumes the target is flat. Curved surfaces (curved cards, posters on curved walls) will cause matrix drift |
| **Scale sensitivity** | Very small targets (< 5cm at arm's length) give insufficient features for reliable lock |
| **Lighting** | Low contrast under poor lighting degrades feature detection significantly |
| **Occlusion** | Partial occlusion of the target causes detection failure — no partial tracking |
| **Distance** | Effective range is roughly 10× the target's physical width. For a 9cm card: ~90cm max distance |
| **Motion blur** | Fast hand movement during scan causes temporary loss |
| **Reflective surfaces** | Glossy card surfaces cause specular highlights that corrupt grayscale features |
| **Symmetric images** | Detection succeeds but orientation can flip 180° — incorrect AR overlay rotation |
| **Single frame detection** | Requires `warmupTolerance` consecutive frames to confirm a lock — adds perceptible latency |

---

## Performance Notes

- `dummyRun()` is important — call it before the first real frame to pre-warm WebGL shader compilation
- TF.js backend selection: defaults to WebGL; falls back to CPU (`@tensorflow/tfjs-backend-cpu`) if WebGL unavailable
- Memory: `.dispose()` the controller when unmounting to release GPU buffers
- Web Worker: compilation runs in a worker; runtime detection/tracking runs on the main thread via TF.js WebGL

---

## Actionable for BizzAR

| Concern | Recommendation |
|---|---|
| AR viewer (`/ar/[userId]/[cardId]`) | Use `maxTrack: 1`, `warmupTolerance: 3` for faster lock feel |
| Jitter reduction | Lower `filterMinCF` to `0.0005`; raise `filterBeta` to `2000` |
| Faster initial lock | Set `warmupTolerance: 1` or `2` — trades false positive risk for snappier UX |
| GPU warm-up | Call `controller.dummyRun()` after `addImageTargets` — before showing the camera preview |
| Multi-target | Not needed for BizzAR's single-card-per-experience model; keep `maxTrack: 1` |
| Cleanup | Always `controller.dispose()` in component `useEffect` cleanup |
| Mobile performance | Test on mid-range Android (Snapdragon 665 class) — TF.js WebGL is the bottleneck |

---

## Sources

- [hiukim/mind-ar-js GitHub](https://github.com/hiukim/mind-ar-js)
- [Tracking Config Docs](https://hiukim.github.io/mind-ar-js-doc/quick-start/tracking-config/)
- [Three.js Image Tracking Docs](https://hiukim.github.io/mind-ar-js-doc/more-examples/threejs-image/)
- [react-three-mind GitHub](https://github.com/tommasoturchi/react-three-mind)
- [react-three-mind npm](https://www.npmjs.com/package/react-three-mind)
- [mind-ar npm package](https://www.npmjs.com/package/mind-ar)
- [webarkit/mind-ar-js fork](https://github.com/webarkit/mind-ar-js)
- [controller.js source](https://github.com/hiukim/mind-ar-js/blob/master/src/image-target/controller.js)
- [matcher.js source](https://github.com/hiukim/mind-ar-js/blob/master/src/image-target/matching/matcher.js)
- [MindAR overview docs](https://hiukim.github.io/mind-ar-js-doc/)
