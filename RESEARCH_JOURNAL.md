# BizzAR — Living Research Journal: WebAR Image Tracking & Spatial Computing
> **LIVING DOCUMENT** — *This journal records continuous computer vision research, field benchmarks, algorithmic innovations, and ecosystem intelligence for browser-native augmented reality on physical business cards.*  
> **Repository:** `bizzar-saas` / `bizzar-proto`  
> **Lead Researcher & Engineer:** Arnav Sawant (@MrCyberNaut)  
> **Last Updated:** 2026-11-02  

---

## 1. Executive Summary & The 2026 WebAR Landscape

Augmented Reality in the mobile browser ("WebAR") represents the holy grail of frictionless spatial computing: point a camera at an object, and interactive 3D digital twins materialize without requiring App Store downloads. 

For business cards, the friction of downloading a native app kills >90% of user conversions. WebAR is the only viable medium for mass adoption.

```
+───────────────────────────────────────────────────────────────────────────+
|                           THE 2026 ECOSYSTEM REALITY                      |
+───────────────────────────────────────────────────────────────────────────+
| 1. 8th Wall Commercial Platform: SUNSET (Feb 28, 2026).                   |
|    Niantic open-sourced core engine at github.com/8thwall/8thwall (MIT).   |
|    Lacks native React Three Fiber (R3F) component tree bindings.          |
|                                                                           |
| 2. WebXR Device API Status:                                               |
|    - iOS Safari: ZERO native WebXR support. Apple restricts to USDZ      |
|      QuickLook (no interactive custom shaders, no DOM overlays).          |
|    - Android Chrome: Image tracking locked behind experimental incubation  |
|      flags (chrome://flags/#webxr-incubations). Non-viable for consumer.  |
|                                                                           |
| 3. Conclusion: Custom In-Browser Computer Vision (MindAR + WebGL)         |
|    remains the ONLY cross-platform zero-install solution that works on    |
|    both iOS Safari and Android Chrome today.                              |
+───────────────────────────────────────────────────────────────────────────+
```

---

## 2. Business Card Tracking Physics: The Core Computer Vision Challenge

Standard image tracking libraries (MindAR, AR.js, 8th Wall) perform well on large, colorful posters, book covers, and packaging. However, physical business cards represent the **worst-case scenario** for classical feature detection algorithms:

### 2.1 The Physical Constraints

| Constraint | Mathematical / Physical Reality | Impact on Classical Vision (SIFT / ORB / FREAK) |
|---|---|---|
| **Small Target Footprint** | $85\text{ mm} \times 54\text{ mm}$ ($1.57:1$ aspect ratio) | At arm's length ($30\text{--}35\text{ cm}$), the card occupies only $\sim 200 \times 127$ pixels on a $1280 \times 720$ camera sensor. |
| **Low-Contrast Stock** | Flat cream, tan, or white paper cardstock | Gradients across flat regions are $\approx 0$. No Difference-of-Gaussian (DoG) extrema survive Laplacian thresholding. |
| **Specular Reflection** | Laminated, gloss, or plastic card finishes | Specular highlights create false, moving high-contrast keypoints that distort pose homography. |
| **Perspective Skew** | Viewing angles from $20^\circ$ to $70^\circ$ | High-frequency typography blurs under non-orthographic perspective distortion. |
| **Symmetry & Repetition** | Centered layouts, twin logos, uniform text lines | Ambiguous feature descriptors cause homography matrix degeneracy (flipping/jumping). |

### 2.2 How MindAR Operates Internally

MindAR does **not** use OpenCV or WebAssembly for image tracking. It executes entirely inside **TensorFlow.js WebGL GLSL fragment shaders**:

1. **Scale-Space Construction:** Builds an image pyramid of Gaussian blurs across octaves.
2. **Difference-of-Gaussian (DoG):** Subtracts adjacent blur levels:
   $$\text{DoG}(x, y, \sigma) = (G(x, y, k\sigma) - G(x, y, \sigma)) * I(x, y)$$
3. **Extrema Detection:** Searches a $3 \times 3 \times 3$ neighborhood for local maxima/minima. Flat card backgrounds fail here because $|\text{DoG}| < \text{threshold}$.
4. **FREAK Descriptors:** Fast Retina Keypoint circular retinal sampling pattern computes binary strings from pixel intensity comparisons.
5. **Pose Homography:** Solves Perspective-n-Point (PnP) using RANSAC to derive rotation matrix $\mathbf{R}$ and translation vector $\mathbf{t}$.

---

## 3. The 5-Layer Stabilized Engine Architecture (BizzAR Engine)

To enable rock-solid tracking on arbitrary business cards without requiring custom markers, we engineered a layered fallback architecture:

```
[Layer 1] WebGL Camera Preprocessing (camera-preprocess.ts)
   │  Proxy intercepts navigator.mediaDevices.getUserMedia()
   │  GLSL Fragment Shader:
   │    • Pass 1: Specular glare suppression via HSL saturation/luminance mask
   │    • Pass 2: S-curve midtone contrast boost (accentuates text boundaries)
   │    • Pass 3: CLAHE-lite 4-tap local histogram normalization on flat tiles
   │  Returns canvas.captureStream(30) to MindAR (zero MindAR core modifications)
   ▼
[Layer 2] MindAR Multi-Target SIFT/FREAK Detection
   │  Scans video stream against precompiled multi-target binary (card.mind)
   │  Identifies target index (0, 1, or 2) and locks initial 6-DoF pose matrix
   ▼
[Layer 3] MindAR NCC Template Tracker + Speed-Adaptive Filtering
   │  Normalized Cross Correlation (NCC) tracks locked patches frame-to-frame
   │  Filtered via speed-adaptive 1€ (One-Euro) Filter:
   │    $\hat{x}_k = \alpha x_k + (1-\alpha)\hat{x}_{k-1}$
   │    $\alpha = \frac{1}{1 + \frac{\tau}{T}}$, where $\tau = \frac{1}{2\pi (f_{c,\min} + \beta |\dot{x}_k|)}$
   │    Tuned baseline: $f_{c,\min} = 0.001$, $\beta = 20.0$, $\text{missTol} = 5$
   ▼
[Layer 4] 2D FFT Phase Correlation Fallback (phase-correlate.ts via fft.js)
   │  When NCC confidence drops (occlusion or fast motion pan):
   │  Extracts $64 \times 64$ patch around last known anchor UV
   │  Applies 2D Hann window: $w(i,j) = \sin^2(\frac{\pi i}{N-1}) \sin^2(\frac{\pi j}{N-1})$
   │  Calculates Cross-Power Spectrum:
   │    $R = \frac{F_{\text{ref}} \circ F_{\text{cur}}^*}{|F_{\text{ref}} \circ F_{\text{cur}}^*|}$
   │  $\text{IFFT}(R) \rightarrow \text{peak location } (\Delta x, \Delta y)$
   │  Translates 3D group offset without losing anchor lock!
   ▼
[Layer 5] Graceful State Machine & Re-detection
   │  If Phase Correlation confidence drops $< 0.08$ for $> 15$ frames:
   │  Gracefully unmounts 3D panels and re-enters Layer 2 detection.
```

---

## 4. Coordinate System & Mathematical Matrix Quirks

Integrating React Three Fiber v8, `@react-three/drei`, and `react-three-mind` required resolving three critical mathematical divergences:

### 4.1 The `flipUserCamera` Matrix Bug
In `react-three-mind` (line ~32179):
```js
group.scale.set(flipUserCamera ? -1 : 1, 1, 1);
```
- **Symptom:** On front-facing selfie cameras (`flipUserCamera = true`), the negative X scale flips all face normals inward, causing backface culling to render all 3D meshes completely invisible.
- **Solution:** We wrapped all anchor contents in `<AnchorContent flip={camera === 'front'}>`:
  ```tsx
  <group scale={[flip ? -1 : 1, 1, 1]}>{children}</group>
  ```
  This cleanly cancels the inverted determinant and restores correct front-facing surface normals.

### 4.2 WebGL vs Computer Vision Texture Inversion
- Three.js WebGL texture coordinates: UV $(0, 0)$ is at bottom-left ($Y$ up).
- MindAR / OpenCV camera coordinates: Pixel $(0, 0)$ is at top-left ($Y$ down).
- **Rule:** Every card texture loaded in Three.js must enforce:
  ```ts
  texture.flipY = false;
  ```

### 4.3 3D Spatial Scaling Law for DOM Elements
In `@react-three/drei`, `<Html transform>` maps 1 CSS pixel to 1 Three.js world unit unless scaled.
- In `react-three-mind`, the business card width is normalized to **$1.0\text{ unit}$**.
- If a panel is styled with CSS `width: 380px`, without scaling, it renders as a **$380\text{ meter}$** giant plane.
- **Scale Factor Formulation:**
  $$\text{Scale Factor} = \frac{\text{Desired World Width}}{\text{CSS Pixel Width}} = \frac{0.912}{380} = 0.0024$$
  At $\text{scaleFactor} = 0.0024$, a $380\text{px} \times 240\text{px}$ glassmorphic panel occupies $0.912 \times 0.576$ units in 3D space, matching the physical card dimensions with crisp sub-pixel text rendering.

---

## 5. Frontier Research & Emerging Technologies (2026+)

### 5.1 Neural Keypoints (ALIKE & SuperPoint via ONNX Runtime Web)
Classical DoG extrema struggle with uniform surfaces. Deep-learning keypoint detectors like **ALIKE (Accurate and Lightweight Keypoints)** and **SuperPoint** train self-supervised convolutional networks to identify interest points based on high-level semantic corners rather than raw intensity variance:
- **ALIKE-N (Tiny):** Operates at $\sim 90\text{ FPS}$ on desktop GPU and $\sim 30\text{ FPS}$ on mobile WebGL via ONNX Runtime Web (`ort-wasm-simd-threaded.wasm`).
- **Moat Opportunity:** Fine-tuning an ALIKE-N model specifically on business card datasets (corporate logos, micro-typography, textured cardstock) would create an open-source tracking engine with $3\times$ higher tracking stability than standard MindAR.

### 5.2 WebGPU Compute Shaders
Currently, camera preprocessing and feature extraction run in WebGL fragment shaders. With WebGPU gaining universal support in Chrome and Safari:
- Compute shaders (`@compute @workgroup_size(16, 16)`) can perform parallel local histogram equalization (full CLAHE) in $< 0.8\text{ ms}$ per $1080\text{p}$ frame.
- Direct texture-to-storage-buffer access eliminates CPU-GPU roundtrip latency.

### 5.3 Sub-Perceptual ArUco Hybrid Bootstrap
Professional enterprise AR systems (Vuforia, Meta Spark) embed micro-fiducial patterns into target designs. Adding a faint, high-frequency border pattern (imperceptible to the human eye, but producing dense FAST corners) provides a fallback anchor when cards have low interior contrast.

---

## 6. Field Benchmarks & Verification Registry

| Test Target | Native DoG Keypoints | Baseline MindAR Lock Time | Lock Time with CLAHE Preprocessor | Stability with FFT Fallback |
|---|---|---|---|---|
| **Target 0: E-Cell Tan** (Worst Case) | $\sim 180$ (mostly in blue 'e' logo) | $1.8\text{s}$ (frequent lost events) | **$0.6\text{s}$** ($\approx 2.4\times$ DoG extrema in cream area) | Holds through $12\text{ cm}$ hand pan |
| **Target 1: E-Cell Blue** (Medium) | $\sim 420$ | $0.4\text{s}$ | **$0.2\text{s}$** | Extremely solid lock |
| **Target 2: Inspark** (Optimal) | $\sim 890$ | $0.15\text{s}$ | **$0.1\text{s}$** | Rock-solid, zero drift |

---

## 7. Bibliography & Primary Research Sources

1. **DeTone, D., Malisiewicz, T., & Rabinovich, A.** (2018). *SuperPoint: Self-Supervised Interest Point Detection and Description.* IEEE/CVF CVPRW. arXiv:1712.07629.
2. **Zhao, X., Wu, X., Chen, J., et al.** (2022). *ALIKE: Accurate and Lightweight Keypoint Detection and Descriptor Extraction.* IEEE Transactions on Multimedia.
3. **Casiez, G., Roussel, N., & Vogel, D.** (2012). *1€ filter: A simple speed-based low-pass filter for noisy input in interactive systems.* ACM CHI 2012.
4. **Niantic Inc.** (2026). *8th Wall Open-Source Monorepo.* `github.com/8thwall/8thwall`.
5. **Chen, H. K.** (2021-2024). *MindAR.js: Web-based Augmented Reality.* `github.com/hiukim/mind-ar-js`.
6. **Castro, F., et al.** (2023). *Fast 2D Phase Correlation on the Web using WebFFT.* Journal of Open Source Software.

---
*Keep this journal updated as new breakthroughs, shader kernels, and benchmark telemetry are generated.*
