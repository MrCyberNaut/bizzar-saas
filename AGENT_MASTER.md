# BizzAR — Master Agent Architecture & State File (AGENT_MASTER.md)
> **FOR AGENTS ONLY: Antigravity & Claude Code**  
> *This document is the single source of truth for all AI agents working on BizzAR. Update this file at the end of every work session.*  
> **Last Synchronized:** 2026-11-01  
> **Repository Owner:** @MrCyberNaut (Arnav Sawant)  
> **Current Focus:** Core AR Engine & Business Card Experience (SaaS Layer is Parked)

---

## 1. Multi-Repository Blueprint & Mapping

The BizzAR ecosystem is organized into **three distinct Git repositories** under the workspace root `D:\Projects\BizzAR-SaaS`:

```
D:\Projects\BizzAR-SaaS\
├── .git/                      ← [REPO 1: ROOT] bizzar-saas (Docs, Knowledge, Master Context)
├── bizzar/                    ← [REPO 2: SAAS] bizzar (Next.js 14, Supabase, Stripe, DnD Editor) [PARKED]
│   └── .git/
├── bizzar-proto/              ← [REPO 3: AR ENGINE] bizzar-proto (MindAR, R3F, WebGL Shaders, Modules) [ACTIVE]
│   └── .git/
├── knowledge/                 ← 30 Research Papers & Technical Deep Dives (00-29 + PDFs)
├── AGENT_MASTER.md            ← THIS FILE (Agent memory & sync across Antigravity + Claude Code)
├── PROJECT_EXPLAINER.txt      ← Plain-text overview for human developer
├── PROJECT_EXPLAINER.html     ← Interactive visual dashboard for human developer
├── CLAUDE.md                  ← Claude Code entrypoint (redirects to AGENT_MASTER.md)
├── JOURNEY.md                 ← Chronological engineering log & bug tracking
├── TODO.md                    ← Phased roadmap & milestone checklist
└── ERROR_LOG.md               ← Error registry & resolution history
```

### Git Repository Status Matrix

| Repository | Path | Remote URL | Active Branch | Purpose & Status |
|---|---|---|---|---|
| **Root (Docs)** | `.` | `https://github.com/MrCyberNaut/bizzar-saas.git` | `master` | **ACTIVE**: Knowledge base, research corpus, agent memory, master documentation. |
| **AR Engine** | `bizzar-proto/` | `https://github.com/MrCyberNaut/bizzar-proto.git` | `feat/mindar-ar-engine` | **CORE FOCUS**: R3F + MindAR runtime, WebGL preprocessing shaders, FFT phase correlation fallback, 3D glassmorphic panels. |
| **SaaS Platform** | `bizzar/` | `https://github.com/MrCyberNaut/bizzar.git` | `master` | **PARKED**: Full-stack Next.js 14 SaaS (Supabase, Stripe, 3D Hub Editor). Parked until AR tracking on cards is flawless. |

> **IMPORTANT GIT RULE:** The root `.gitignore` ignores `bizzar/` and `bizzar-proto/`. Each sub-repository manages its own git history and remote. Always run `git -C bizzar <command>` or `git -C bizzar-proto <command>` when inspecting or committing within the sub-projects.

---

## 2. Core Architectural Pivot: 8th Wall → MindAR + R3F

1. **Why 8th Wall Was Abandoned:**
   - 8th Wall shut down on **February 28, 2026**.
   - The initial SaaS viewer in `bizzar/src/app/ar/[userId]/[cardId]/route.ts` used 8th Wall OSS + A-Frame script tags. This is obsolete.
2. **The Modern Target Engine (`bizzar-proto`):**
   - **Framework:** Next.js 14 (App Router) + React Three Fiber v8 (`@react-three/fiber`) + Drei (`@react-three/drei`).
   - **Tracking Runtime:** `mind-ar` (v1.1.5) wrapped via `react-three-mind` (v0.3.0).
   - **Computer Vision Reality:** MindAR does **NOT** use ORB or WASM. It uses **Difference-of-Gaussian (DoG) keypoint detection + FREAK binary descriptors** executed entirely inside **TensorFlow.js WebGL GLSL shaders**.
   - **The Business Card Problem:** Business cards with uniform background areas (e.g. flat cream/tan cardstock) lack high-frequency gradients. The DoG extrema do not pass the Laplacian threshold, causing frequent tracking loss.

---

## 3. The Layered AR Engine Pipeline (`bizzar-proto`)

To track arbitrary physical business cards without requiring custom markers, we engineered a 5-layer fallback architecture:

```
[Layer 1: WebGL Camera Preprocessing] (camera-preprocess.ts)
   │  Intercepts navigator.mediaDevices.getUserMedia() via proxy
   │  GLSL Fragment Shader: Glare suppression + S-curve midtone contrast + CLAHE-lite
   │  Returns canvas.captureStream(30) to MindAR (zero MindAR core modifications)
   ▼
[Layer 2: MindAR DoG / FREAK Keypoint Detector]
   │  TF.js WebGL kernels search for scale-space extrema across image pyramid
   │  Matches against precompiled multi-target .mind binary
   ▼
[Layer 3: MindAR NCC Template Tracker]
   │  Normalized Cross Correlation per-frame tracking of locked keypoints
   │  Stabilized with One-Euro Filter (minCF: 0.001, beta: 20, missTol: 5)
   ▼
[Layer 4: FFT Phase Correlation Fallback] (phase-correlate.ts)
   │  When MindAR loses lock (confidence drops below threshold):
   │  Extracts 64x64 video patch around last-known anchor UV
   │  Applies 2D Hann window -> 2D FFT -> Cross-Power Spectrum -> IFFT
   │  dx, dy peak translation keeps 3D panels anchored through brief occlusions
   ▼
[Layer 5: Re-detect State Machine]
   │  If Phase Correlation confidence drops < 0.1 for > 15 frames:
   │  Drop anchor and re-enter MindAR L2 detection
```

---

## 4. Critical Mathematical & Coordinate System Rules

When working in `bizzar-proto/src/components/ar/`:

### A. The `react-three-mind` Inversion Bug (`AnchorContent.tsx`)
`react-three-mind` internally applies `scale={[flipUserCamera ? -1 : 1, 1, 1]}` at line ~32179.
- On front-facing camera, this flips the X-axis, turning mesh normals inside-out and rendering 3D content invisible.
- **Fix:** `<AnchorContent flip={camera === 'front'}>` explicitly wraps all children and applies compensatory inverse scaling:
  ```tsx
  <group scale={[flip ? -1 : 1, 1, 1]}>{children}</group>
  ```

### B. Texture Y-Flip
MindAR uses standard computer vision coordinate space (origin at top-left, Y pointing down). Three.js uses WebGL space (origin bottom-left, Y pointing up).
- **Rule:** Every card image texture loaded via `useTexture` MUST set:
  ```ts
  texture.flipY = false;
  ```

### C. Performance & Mobile R3F Rules
- Target device floor: **Mid-range Android (30–60fps)**.
- R3F Canvas config:
  - `frameloop="always"` (mandatory for AR video stream synchronization).
  - `dpr={[1, 1.5]}` (never exceed 1.5 on mobile to avoid GPU thermal throttling).
  - Direct DOM mutation in `useFrame()` for FPS/pose text (never trigger React re-renders in animation frame loops).
- Preprocessor frame throttle:
  - `camera-preprocess.ts` caps shader loop at 30fps (`1000 / 30 ms`) on mobile to conserve GPU cycles for R3F.

---

## 5. 3D Modular Glassmorphic Panel System

Located in `bizzar-proto/src/components/`:
- **`Panel.tsx`**: Floating glassmorphic 3D container using `@react-three/drei`'s `<Html transform>`.
- **Panel Slots & Spatial Transforms**:
  - `overlay`: Position `[0, 0, 0.001]`, flat against card surface (pulsing outline, scan lines, `CardOverlay.tsx`).
  - `top`: Position `[0, 0.82, 0.05]`, tilted backward (vCard download, `ContactCard.tsx`, Bio).
  - `left`: Position `[-0.85, 0.2, 0.1]`, angled inward (Social links, GitHub contribution graph).
  - `right`: Position `[0.85, 0.2, 0.1]`, angled inward (Image gallery, 3D model, Video embed).
- **Available Modules**:
  - `Bio.tsx`: Professional summary with DM Sans typography.
  - `VCard.tsx`: One-tap "Add to Contacts" (.vcf generation with mobile haptic feedback).
  - `Socials.tsx`: Interactive links (Twitter/X, LinkedIn, GitHub).
  - `GitHubGraph.tsx`: Live SVG commit heatmaps.
  - `ImageGallery.tsx`: Project / portfolio image carousel.
  - `VideoPanel.tsx`: Responsive video embed.
  - `Model3D.tsx`: Interactive 3D asset preview.
  - `CardOverlay.tsx`: Animated border glow + scan-line effect.

---

## 6. Multi-Card Target Inventory & Compilation

Located in `bizzar-proto/public/`:
- Multi-target binary: `card.mind` (973 KB, contains 3 trained targets).
- Registered Targets in `bizzar-proto/src/lib/ar-config.ts`:
  - **Index 0:** `E-Cell (tan)` (`/card1.jpg`) — MIT E-Cell tan card.
  - **Index 1:** `E-Cell (blue)` (`/card2.jpg`) — MIT E-Cell blue logo card.
  - **Index 2:** `Inspark` (`/card3.png`) — Inspark event card (high keypoint density).
- **Compilation Tooling:**
  - Headless script: `bizzar-proto/scripts/compile-mind.js` (uses Playwright Chromium).
  - Web UI compilers: `bizzar-proto/public/compiler.html` and `compiler-auto.html`.

---

## 7. Immediate Roadmap: Getting Main AR Rock-Solid

Our North Star right now is: **Make the AR business card experience look breathtaking, track effortlessly, and feel engineered like an Apple product.** The SaaS platform (auth, stripe, database) is deferred until this engine is production-grade.

### Sprint Checklist:
1. **Engine Polish:**
   - [x] WebGL camera preprocessing (glare reduction + contrast boost).
   - [x] CLAHE-lite local histogram normalization in fragment shader.
   - [x] Phase correlation tracking recovery via `fft.js`.
   - [ ] Field validation on all 3 target cards across lighting environments.
   - [ ] Smooth spring transitions when switching from tracking to recovered state.
2. **Product UI / UX:**
   - [ ] Clean, hyper-aesthetic mobile launch UI (minimalist HUD, scan bracket guide, detection feedback).
   - [ ] Haptic feedback (`navigator.vibrate`) on anchor acquisition.
   - [ ] Mount active panel components (`Bio`, `Socials`, `GitHubGraph`) in 3D around the detected card.
   - [ ] One-tap vCard contact save with visual success confirmation.
3. **Future (SaaS Layer Re-activation):**
   - Connect Supabase card CRUD to dynamic `.mind` compilation and live R3F AR route.

---

## 8. Agent Operating Rules (Claude Code & Antigravity)

1. **Model Selection:**
   - Architecture & Strategy: Top reasoning models (Opus / Gemini Pro).
   - Code Modifications & Feature Implementation: Sonnet / Gemini Flash (High).
   - Documentation & Logging: Fast models (Haiku / Flash).
2. **Error Logging Protocol:**
   - Any runtime, compilation, or tracking bug MUST be immediately registered in `d:\Projects\BizzAR-SaaS\ERROR_LOG.md`.
   - Once resolved, record the exact root cause and fix in `ERROR_LOG.md`.
3. **Session Close Checklist:**
   - Before completing any session:
     1. Verify code compiles (`npm run build` in active repo).
     2. Update `AGENT_MASTER.md` with new findings, architecture changes, or state updates.
     3. Update `PROJECT_EXPLAINER.txt` and `PROJECT_EXPLAINER.html` for the user.
     4. Log session progress in `JOURNEY.md`.
     5. Commit and push changes to the relevant Git repository.
