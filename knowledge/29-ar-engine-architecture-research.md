# AR Engine Architecture Research — Deep Dive
Source: MindAR source archaeology + web research, 2026-05-22

## Mental Model Correction
MindAR does NOT use ORB. It uses a SIFT-family pipeline (DoG + FREAK descriptor),
all implemented as custom TF.js WebGL GLSL kernels. No WASM anywhere.

The failure on low-texture cards is not a weak detector — it's that flat regions
genuinely have no DoG extrema that survive the Laplacian threshold (9.0) and
Hessian edge rejection. The pipeline is correct. The card texture is the constraint.

## MindAR Full Pipeline (from source)

### Offline (compilation)
1. Greyscale + multi-scale pyramid (image-list.js)
2. Gaussian pyramid via BinomialFilter WebGL kernel (4th-order binomial, approx Gaussian)
   — THIS is what crashes in headless: needs GPU for BinomialFilter kernel
3. DoG pyramid (image1 - image2, tensor subtraction)
4. BuildExtremas GLSL: 3×3×3 extrema + Laplacian threshold (9.0) + Hessian edge reject (25/4)
5. Sub-pixel localization (ComputeLocalization kernel → CPU quadratic fit)
6. Orientation assignment (ComputeOrientationHistograms → 36 bins → peak)
7. FREAK descriptor: 37 sample points, 666 pairwise comparisons → 672-bit binary descriptor
   (NOT ORB/BRIEF — FREAK from 2012, more rotation-invariant)
8. k-medoids hierarchical clustering (128 hypotheses, 8 centers/node, Hamming distance)
   — This is the vocabulary tree for fast runtime matching
9. Tracking data: 2-scale (256px, 128px) with CPU Sobel gradient + NCC distinctiveness filter

### Runtime (per frame)
1. Camera frame → greyscale (ITU-R BT.601 GLSL: 0.299r + 0.587g + 0.114b)
2. Detection mode: crop 3×3 grid positions (one per frame), run full SIFT/FREAK detection
3. Matching (Web Worker): k-medoids tree traversal + Lowe's ratio test (threshold 0.7)
4. Hough voting: 4D bin (tx, ty, angle, scale) for geometric consistency pre-RANSAC
5. RANSAC homography: 4-point DLT, Cauchy robustness cost, min 6 inliers
6. Guided matching: use H to constrain search, second RANSAC pass
7. Pose: K_inv * H decomposition → [R|t], then ICP refinement (10 iterations, 6-DOF)
8. TRACKING mode switch: NCC template matching
   - Re-project keyframe via _computeProjection WebGL kernel
   - NCC 13×13 template vs 21×21 search window per feature
   - Reject if NCC score < 0.8 (AR2_SIM_THRESH)
   - ICP refine from NCC correspondences
9. One Euro Filter smoothing (filterMinCF, filterBeta params we tune)

## Key Files in MindAR Source
- detector/detector.js — full SIFT pipeline
- detector/kernels/webgl/ — all GPU kernels (BinomialFilter, BuildExtremas, FREAK, etc.)
- matching/hierarchical-clustering.js — k-medoids tree build
- matching/matching.js — Lowe's ratio test + guided matching
- matching/hough.js — 4D Hough voting
- matching/ransacHomography.js — RANSAC + homography solve
- estimation/estimate.js + refine-estimate.js — pose decomp + ICP
- tracker/tracker.js — NCC template tracking (the cheap per-frame path)
- libs/one-euro-filter.js — smoothing

## What AR.js Uses (Different Architecture)
jsartoolkit5 — WebAssembly port of ARToolKit5 (C++ → WASM).
For NFT: KPM (Key Point Matching) based on OpenCV SURF/FREAK inside WASM.
AR.js is a JS wrapper around the WASM blob. Started 2017, marker-only.
MindAR's architecture: pure TF.js + GLSL, no native code.

## Novel Approaches Found (Research)

### Tier 1 — Build now, high ROI

**1. CLAHE-lite WebGL**
- Contrast Limited Adaptive Histogram Equalization as fragment shader
- Subdivide canvas into NxN tiles, run local histogram stretch per tile
- Reference: fivekogfx (github.com/fiveko/fivekogfx) — JS+GLSL histogram equalization
- Build on camera-preprocess.ts (already in bizzar-proto)
- Expected: ~2× more DoG extrema on flat-background cards

**2. Micro ArUco/AprilTag hybrid (optional fast-path)**
- Library: js-aruco2 (github.com/damianofalcioni/js-aruco2) — pure JS, no deps
- Also: apriltag-js-standalone (faster, same interface)
- ~3–10 FPS for detection but only needed once at initial lock
- Card owners can embed a tiny marker styled as a design element
- After lock: hand off to NCC tracking (no marker needed frame-to-frame)
- This combination (fiducial bootstrap → NFT handoff) is NOT implemented publicly
  → Potential open source contribution

### Tier 2 — Medium effort, high payoff

**3. FFT Phase Correlation for tracking stability**
- After MindAR initial lock, use FFT for frame-to-frame translation tracking
- Library: WebFFT (webfft.com) — auto-selects best FFT per browser, WebSIMD when available
- Library: PulseFFT (github.com/AWSM-WASM/PulseFFT) — WebAssembly kissFFT port
- 2025 Nature paper directly addresses phase correlation on featureless images:
  https://www.nature.com/articles/s41598-025-94176-x
- Architecture: detect slowly (MindAR), track fast (phase correlation), re-detect if lost

**4. Card edge detection bootstrap (OpenCV.js)**
- OpenCV.js already has Canny, findContours, homography
- Canny → quadrilateral fit → homography → AR pose
- Completely texture-independent initialization
- ~200 lines of glue code
- Challenge: disambiguation from other rectangles; mitigate with aspect ratio + N-frame stability

### Tier 3 — Research-grade, not browser-ready

**5. Line feature fusion (PLFG approach)**
- PLFG-SLAM (ScienceDirect 2025): point-line feature fusion
- Line features (text baselines, card borders) are abundant where corner points are sparse
- EDlines extraction + graph neural network matching
- No browser implementation exists

**6. Deep feature extractors (SuperPoint/ALIKE/DISK)**
- ALIKE-N: 318KB model, 95fps on desktop GPU
- Still needs: ONNX export → onnxruntime-web with WebGL backend
- No one has done this for image tracking (research contribution)
- Timeline: 2-4 weeks of focused work

## The Detect → Track Split (Key Architectural Pattern)
Professional AR SDKs (8th Wall, Snap) do NOT re-detect features every frame.
They detect ONCE (slow, can take multiple frames), then track cheaply per-frame.
MindAR's NCC tracker already does this — but only after initial SIFT lock.
The weakness: if NCC drops below 0.8, it falls all the way back to full SIFT detection.

Better architecture:
  Any card scanned
    → [L1] CLAHE WebGL preprocessing (more features from flat regions)
    → [L2] MindAR initial lock (SIFT/FREAK, may take 2-5 frames)
    → [L3] NCC template tracking (MindAR built-in, cheap)
    → [L4] Phase correlation fallback (if NCC drops, before full re-detect)
    → back to L2 if L4 fails

No fiducial required. ArUco optional fast-path for cards that have it.

## What BizzAR Should Build (Recommendation)
Solve for ANY card (not just BizzAR-designed). Rationale:
- The BizzAR pitch is "your existing card comes alive" — requiring custom printing breaks this
- Most real cards (logos, typography, color blocks) have enough texture with CLAHE boost
- The e-cell tan card is worst-case; typical cards track fine
- Arbitrary tracking is the product moat; fiducial tracking is a printing service

## Practical Replaceable Modules in MindAR
- Feature detector: replace detector.js SIFT → ALIKE-N via ONNX (future)
- Descriptor: replace FREAK → learned binary descriptor (future)
- Tracking: augment NCC with phase correlation fallback (near-term)
- Preprocessing: CLAHE before MindAR sees the feed (near-term, in camera-preprocess.ts)
- Matching: add WASM SIMD for Hamming distance in matching/hamming-distance.js
- ICP: hand-unroll the 6×6 solve instead of ml-matrix inversion (performance)

## Sources
- MindAR source: d:/Projects/BizzAR-SaaS/bizzar-proto/node_modules/mind-ar/src/image-target/
- js-aruco2: https://github.com/damianofalcioni/js-aruco2
- fivekogfx: https://github.com/fiveko/fivekogfx
- WebFFT: https://webfft.com/
- PulseFFT: https://github.com/AWSM-WASM/PulseFFT
- Phase correlation 2025: https://www.nature.com/articles/s41598-025-94176-x
- PLFG-SLAM: https://www.sciencedirect.com/science/article/abs/pii/S0263224125017944
- Adaptive ORB threshold: https://www.researchgate.net/publication/395446076
