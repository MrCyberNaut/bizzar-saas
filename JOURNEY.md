# BizzAR — Project Journey & Engine Documentation
**Started:** 2026-05-21  
**Status:** Active — proto engine phase  
**Last updated:** 2026-05-21

> This document is a living record. Update it every session. It is the brain of this project — technical decisions, dead ends, discoveries, what's next.

---

## 1. What We Are Building — The Full SaaS Vision

**BizzAR** turns a physical business card into a live AR experience, scannable by anyone with a phone, no app required.

### The user flow in one sentence
Professional uploads card → drags modules into a floating 3D panel editor → publishes → anyone who receives the card points their phone at it and the panels animate into view, anchored to the card in real space.

### The product layers
```
Physical card (trigger)
    ↓  camera sees it
AR Engine (bizzar-proto) — image tracking + R3F scene
    ↓  anchor detected
Panel system — 4 floating panels around the card (overlay, top, left, right)
    ↓  content
Module system — vCard, Socials, Bio, GitHub graph, Image Gallery, Video, Calendly
    ↓  gated by plan
Freemium — Free: overlay + top panels. Pro: all 4 + pro modules
    ↓  published as
Web URL + QR code — no app install, opens in mobile browser
```

### Stack (final SaaS target)
| Layer | Tech |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind |
| 3D Editor | React Three Fiber + Drei (orthographic editor) |
| AR Runtime (proto/R&D) | react-three-mind + MindAR (current active work) |
| AR Runtime (SaaS target) | TBD — see §4 |
| State | Zustand |
| Backend | Supabase (Postgres, Auth, Storage, Edge Functions) |
| Payments | Stripe |
| Drag & Drop | @dnd-kit/core |

### Business context
- Digital business card market: **$215M in 2025**, growing to **$680M by 2035** (12.2% CAGR)
- 25–30% of professionals prefer digital over paper cards. Tech sector already >50%
- AR adds the wow factor that NFC/QR alone doesn't deliver
- 8th Wall (formerly the industry standard) **shut down February 28, 2026** — the field is wide open
- No competitor does browser-native AR + drag-drop module editor + physical card trigger in one product
- Closest competitors: Artivive (artist-focused), AR Code (QR-trigger, no module editor), HoloCardz (custom dev shop, not SaaS), Blippar (enterprise pricing)

---

## 2. What the Proto Engine (`bizzar-proto`) Was Supposed to Prove

`bizzar-proto` is not a toy — it is the R&D environment for the AR engine that will power the SaaS viewer. The prototype must prove:

- [ ] **Image tracking is stable enough** for real-world business cards (not lab conditions)
- [ ] **React Three Fiber + MindAR** can power the module panels without fighting each other
- [ ] **The AnchorContent coordinate system** is solvable (front vs back camera flip)
- [x] WebGL camera preprocessing pipeline intercepts getUserMedia without breaking MindAR
- [x] Glare reduction improves ORB feature point detection on reflective cards
- [x] Debug tooling (FPS counter, pose display, filter profiles, event counters) is in place
- [ ] Overlay alignment is stable and correct from all camera angles
- [ ] Panel content (vCard download, etc.) is fully functional in AR space
- [ ] Performance is 60fps on mid-range Android (the worst target device)

**What has NOT been built yet in the proto:**
- Left, right, top panel geometry (only overlay/contact card is present)
- Module rendering system inside panels
- Any content beyond the vCard download button
- AR viewer that matches the SaaS data model (fetches from Supabase)
- Target image quality analysis tool (to know if a user's card is trackable before publishing)

---

## 3. Where We Are in the Proto Pipeline

### Done
- **Stack wiring**: react-three-mind + R3F + Next.js 14, SSR=false, COOP/COEP headers
- **MindAR .mind compiler** (`/compiler.html`) — drop image, download `.mind` file
- **AnchorContent flip fix** — identified undocumented `scale=[-flipUserCamera,1,1]` in react-three-mind line ~32179; built wrapper that cancels it conditionally per camera mode
- **onReady infinite loop fix** — inline callbacks caused re-renders → ARView re-init loop; fixed with `useCallback` + stable `add()` ref
- **texture.flipY=false** — MindAR coordinate space has inverted Y vs image space
- **Camera switcher** — back/front toggle with ARView remount via `key` prop
- **WebGL preprocessing pipeline** — getUserMedia interceptor → GLSL fragment shader (glare suppression + midtone contrast boost) → `canvas.captureStream(60)` returned to MindAR
- **Bypass uniform** — glare toggle sets `u_bypass` float uniform, no stream restart
- **Filter profiles** — SMOOTH / BALANCED / FAST presets, ARView remounts on change
- **FPS counter** — direct DOM mutation in useFrame, console warns if <30fps
- **Debug panel** — status, fps, pose (pos+rot+scale), found/lost counters, profile buttons, glare toggle, reset

### Currently broken / in-progress
- **Overlay alignment** — tracking works but overlay direction/jitter is still off; the AnchorContent flip fix for back camera was just applied, needs mobile re-test
- **Scale readout** showing ~1599** — this is the `matrixWorld.decompose()` reading the full camera-space magnitude; it is not a bug per se but the number is confusing — needs a cleaner world-relative display
- **Tracking frequency** — ORB struggles with the e-cell card (tan/uniform background area); see §4 for solutions

### Debug state (as of last session)
```
Profile: BALANCED (minCF=0.001, β=20, miss=5, warmup=3)
Status: sometimes reaches TRACKING but jitters and loses frequently
AnchorContent: flip={camera==='front'} — just fixed, untested
Glare filter: ON by default, bypass toggle working
FPS: not yet measured post-fix
```

---

## 4. What Is Needed to Make This Engine Actually Work

This is the hard part. We are doing something almost nobody has done before: **a production-quality, browser-native, image-tracking AR engine for physical business cards with a SaaS wrapper**. Here's what stands between us and that.

### 4.1 The Core Tracking Problem

**ORB (Oriented FAST and Rotated BRIEF)** — MindAR's feature detector — is a classical computer vision algorithm. It finds corners and edges by looking for high-contrast gradients. Business cards are its worst enemy:

| Card property | Why it breaks ORB |
|---|---|
| Uniform background (tan/cream area) | Zero gradient = zero keypoints |
| Glossy/reflective lamination | Specular highlights = false bright spots |
| Symmetrical layout | Identical features = ambiguous matching |
| Small physical size | Few pixels per feature at arm's length |
| White edges | Border region is high-gradient but non-unique |

**What MindAR actually needs from a target image:**
- 300–1000 unique, well-distributed keypoints
- High contrast at corners (not gradients — sharp steps)
- No repeated/symmetrical regions
- Asymmetric layout (the tracking system needs to know which end is which)

**The e-cell card**: has the blue "e" logo area (good) + tan left half (bad). The logo area carries almost all the keypoints. This is why tracking breaks when the logo is partially occluded or at a steep angle.

### 4.2 What We Can Do in the Engine

**Tier 1 — Already done / in flight**
- WebGL glare preprocessing (done ✓)
- One Euro Filter tuning via profiles (done ✓)
- missTolerance + warmupTolerance tuning (done ✓)

**Tier 2 — Next engineering priorities**

**A. Adaptive contrast enhancement in the shader**
Current shader applies uniform contrast boost. What we actually need: CLAHE-style local contrast normalization — divide frame into tiles, equalize each tile's histogram independently. This dramatically helps uniform-background targets by amplifying micro-texture that ORB ignores at global scale.
```
Research needed: CLAHE in GLSL (computable per-tile in a fragment shader)
```

**B. Target image quality analyzer**
Before a user publishes their card, run the card image through a JS port of ORB keypoint detection and visualize the keypoint distribution. Show them a heatmap and a score. If score < threshold, warn them and suggest: "Your card has a low-texture area on the left. Consider adding a graphic element, or use our high-contrast border template."

This is a UX feature that prevents the most common support request: "my AR doesn't work."

**C. Per-card `.mind` compilation quality gate**
When the server compiles `.mind` from the uploaded card image, analyze the number of keypoints. If < 200, flag the card and show the user a warning in the dashboard with suggestions.

**D. Fallback tracking: sparse optical flow on top of ORB**
MindAR uses only ORB for both detection and tracking. After initial detection, optical flow (Lucas-Kanade) would be far more stable for tracking already-found features frame-to-frame. MindAR's one-euro filter smooths the output but doesn't help with lost features between ORB frames.

This would require either:
- A PR to MindAR (open source contribution opportunity)
- A wrapper that runs our own LK flow on the canvas between MindAR's keyframe updates

**E. ML-based feature detector (long game)**
SuperPoint (2018, Magic Leap) is a self-supervised neural net that produces far more keypoints on low-texture surfaces than ORB. WebAR.rocks has proven neural nets can run at 60fps in the browser via WebGL. The question is whether a lightweight SuperPoint can be ported to WASM/WebGL for use as MindAR's detector. This would be a genuine research contribution — nobody has done this for web AR image tracking.

**F. ArUco fiducial markers as a hybrid trigger**
Add an invisible (sub-perceptual to humans) ArUco-style pattern into the card design template. This is a high-keypoint pattern that ORB loves, giving the tracker an anchor even when the rest of the card is low-texture. This is what professional AR systems (Vuforia, Metainfo) do with their target images.

### 4.3 Coordinate System — The AnchorContent Problem

**What we discovered (original):**
react-three-mind wraps every ARAnchor in `scale={[flipUserCamera ? -1 : 1, 1, 1]}` at line ~32179. This is undocumented. When `flipUserCamera=true` (front camera), all mesh normals face inward → invisible content.

**What we fixed:**
`AnchorContent` wrapper applies matching scale to cancel it. Pass `flip={camera === 'front'}`.

**What still needs validation:**
Does the overlay align correctly from multiple camera angles (steep angle, shallow angle, rotated)? The pose display (pos, rot, scale) in the debug panel is now in world space — we need to validate these numbers make geometric sense against the physical card position.

**Open question:**
MindAR outputs camera-relative pose. react-three-mind transforms this into R3F world space. The transformation chain is:
```
MindAR raw pose → react-three-mind world transform → ARAnchor group → AnchorContent → our content
```
We don't fully understand the react-three-mind world transform step. Debugging this requires reading the react-three-mind source and comparing pose output with a known physical setup (card at known distance/angle).

### 4.4 Performance on Real Devices

Target device: **mid-range Android** (2020–2022 era, ~$200 price point). This is the median phone in India/SEA, the core market.

Known constraints:
- Single WebGL context: MindAR's tracking + our shader pipeline share GPU time
- Mobile Safari (iOS) has WebGL limitations and no `captureStream` — our preprocessing pipeline won't work on iOS
- 60fps is aspirational; 30fps is the real floor for usable AR

**What we need to know:**
- Does `canvas.captureStream(60)` on Android Chrome actually deliver 60fps?
- What is the actual FPS drop from our shader pipeline vs raw MindAR?
- Can we detect iOS and disable the preprocessing pipeline automatically?

---

## 5. What We Are Doing Right and Wrong

### Doing Right
- **Build order**: proto first, SaaS on top. Not reinventing — lifting from proto.
- **R3F over A-Frame**: The SaaS module system (panels, widgets, animations) would be impossible to build cleanly in A-Frame's HTML entity model. R3F gives us the full component tree, Suspense, context, and Three.js primitives.
- **Debug-first**: The debug panel, console logs, and pose display are the right call. We can't fix what we can't see. The panel is temporary but invaluable now.
- **getUserMedia interceptor**: The approach of preprocessing before MindAR sees the stream is architecturally correct. No MindAR modifications needed. Clean separation.
- **Stable callbacks**: Every `onReady`/`onFound`/`onLost` is `useCallback` with stable deps. This was the fix for the infinite re-init loop and it's the right pattern forever.
- **Open source awareness**: The AnchorContent flip, the getUserMedia preprocessing pipeline — both are novel and undocumented. We should blog/PR these.

### Doing Wrong / Risks
- **Reflecting cards**: We keep trying to track a glossy business card. The right answer is to also design a card template that is inherently high-trackability (matte, asymmetric, high-contrast edges). We are solving hardware with software.
- **No iOS testing**: Our entire preprocessing pipeline uses `canvas.captureStream()` which is unavailable in iOS Safari. We need a fallback detection path and must test on iPhone.
- **Scale readout confusion**: `matrixWorld.decompose()` returns camera-space magnitude. The "scale:1599" display has caused confusion — it's not wrong but it's not useful either. Need a better metric.
- **react-three-mind is essentially abandoned**: Last meaningful commit was 2023. Version 0.3.0 (May 2025) is just a dependency bump. We are building on top of an unmaintained binding. The MindAR core (hiukim) is more active but slow. This is a technical debt risk.
- **Not testing the full user flow**: We are tracking a card but the actual product (panels + modules) doesn't exist in the proto yet. We risk solving tracking in isolation and then discovering panel layout breaks everything.

---

## 6. Integration Plan — Proto → SaaS

The proto proves the engine. The SaaS wraps it. Here is the handoff plan:

### Phase A — Engine stabilization (current)
1. Fix overlay alignment (AnchorContent flip validation)
2. Validate tracking stability on multiple cards (high texture, low texture)
3. Achieve stable 30fps on Android mid-range
4. Implement CLAHE shader for better contrast on uniform backgrounds
5. Build target image quality analyzer

### Phase B — Panel system in proto
6. Add the 4 panel geometry (overlay, top, left, right) as R3F planes
7. Wire module rendering into panels (start with vCard, Socials, Bio)
8. Prove the full scene renders at acceptable FPS with panels visible

### Phase C — SaaS viewer
9. Build `/ar/[userId]/[cardId]` Next.js route that fetches card + panel + module config from Supabase
10. Replace hardcoded `CARD_CONFIG` with Supabase data
11. Compile `.mind` files server-side (Edge Function) on card upload
12. Remove all debug UI

### Phase D — Editor
13. Build the R3F isometric editor (`/editor/[cardId]`)
14. Panel slot system with dnd-kit module placement
15. Module config forms (vCard, links, etc.)

### Phase E — Auth + payments
16. Supabase Auth (magic link / Google)
17. Stripe checkout + webhook → `users.plan = pro`
18. RLS enforcement on panels + modules

---

## 7. Ideas We Want to Explore

### Immediate / high value
- **Card scan quality feedback UI**: Before publishing, show user a keypoint heatmap of their card with a color score. Green = trackable. Red = will struggle. Yellow = marginal. Let them crop/zoom to improve.
- **"BizzAR badge" watermark for free tier**: Every AR view from a free account shows a small "Made with BizzAR" badge in the corner. Acts as organic growth.
- **Sparkles on anchor found**: `<Sparkles>` from drei, 5 lines, fires on `onAnchorFound`. Huge emotional payoff for first-time scan.
- **Haptic feedback** on card detection (already done, `navigator.vibrate(50)`).

### Medium term
- **CLAHE fragment shader**: Local contrast normalization per tile. Better than global glare suppression for uniform backgrounds.
- **Card design templates**: Provide a set of "AR-optimized" card templates in the editor — pre-designed with asymmetric, high-contrast layouts that score well in the keypoint analyzer.
- **Multi-target support**: One AR experience triggered by different cards (front/back, different cards for different team members all under one account).
- **Video as overlay**: Looping video clip anchored to the card surface — the "living photo" effect. Very high virality potential.

### Long game / research
- **SuperPoint ML detector**: Replace MindAR's ORB detector with a WASM/WebGL port of SuperPoint. Would dramatically improve tracking on low-texture surfaces. This has never been done for web AR. Publishable research + massive competitive moat.
- **LK optical flow hybrid**: After initial ORB detection, switch to Lucas-Kanade sparse flow for per-frame tracking. Far more stable than re-running ORB every frame.
- **Custom `.mind` format enhancements**: The `.mind` file format is open. Could encode more keypoints, store multiple image pyramid levels with different preprocessing, add ArUco markers as a hybrid anchor.
- **Offline-first AR**: Service worker caches the `.mind` file so the AR works in airplane mode after first load. Critical for conference/networking scenarios.
- **AR analytics**: Every scan gets logged (with consent). Creator can see: how many people scanned, what they tapped, geographic distribution. This is a strong Pro feature.

---

## 8. What You Need to Provide — Research & Materials

To properly build this engine and make it competitive, here is what would meaningfully accelerate the work:

### Immediate needs (will unblock current tracking issues)
- **Photos of the e-cell card under different lighting conditions** — overhead fluorescent, angled light, outdoor sun, dark indoor. We can tune the GLSL shader per lighting environment if we know the range.
- **A second test card** — ideally one with dense graphics (logo, photo, many elements). This will help us isolate whether the problem is the engine or the card.
- **Screen recording of the debug panel while scanning** — need to see the FPS counter and pose display together to diagnose the jitter pattern.

### Research papers (would meaningfully inform SuperPoint/CLAHE work)
- **SuperPoint: Self-Supervised Interest Point Detection and Description** (DeTone, Malisiewicz, Rabinovich, 2018) — The paper. arXiv:1712.07629
- **ALIKE: Accurate and Lightweight Keypoint Detection and Descriptor Extraction** (2022) — Lighter than SuperPoint, designed for mobile/edge
- **Homographic Adaptation** — the training technique SuperPoint uses, relevant if we want to fine-tune on business card images
- **CLAHE (Contrast Limited Adaptive Histogram Equalization)** — any GPU/GLSL implementation paper
- **Lucas-Kanade optical flow on GPU** — if we go the hybrid tracking route

### Community intelligence (worth following)
- **r/webdev + r/augmentedreality** — search "MindAR business card" — there are threads with people hitting the same problems
- **@hiukim_dev on Twitter** — the MindAR author. He responds to issues occasionally. Worth DMing about the ORB-on-business-cards limitation.
- **Three.js Discord** (#react-three-fiber channel) — people have solved the AnchorContent/coordinate system issues before
- **WebXR Discord** — the serious web AR community
- **GitHub issues on hiukim/mind-ar-js** — issues #556 (unstable tracking), many others with filter param discussions
- **WebAR.rocks GitHub** — their neural object detector is the closest thing to SuperPoint-in-browser that exists today

### Things to find / dig up
- Any Twitter/X thread from WebAR developers about `canvas.captureStream` + MindAR preprocessing (almost certainly nobody has done this — we may be first)
- Blog posts from Artivive or Zapworks engineers about image target quality — they have the same problem and their engineering blogs may have solved parts of it
- The 8th Wall OSS codebase (now archived) — they had significantly better tracking than MindAR and their image target algorithm may be partially open
- Any WASM port of ORB or SIFT for browsers — might be a drop-in upgrade to MindAR's detector

---

## 9. Known Bugs Log

| # | Bug | Status | Fix |
|---|---|---|---|
| 1 | onReady infinite loop | Fixed | useCallback with stable add() ref |
| 2 | Content invisible (scale -1666) | Fixed | AnchorContent scale=[flip?-1:1,1,1] |
| 3 | Card texture upside down | Fixed | texture.flipY = false |
| 4 | getUserMedia infinite loop in preprocessor | Fixed | start() calls _original directly |
| 5 | Glare toggle kills stream (stuck scene) | Fixed | u_bypass GLSL uniform, no stream restart |
| 6 | Camera feed inverted | Fixed | UNPACK_FLIP_Y_WEBGL = true |
| 7 | 30fps cap on processed stream | Fixed | captureStream(60) |
| 8 | getUniformLocation called every frame | Fixed | Cache in constructor |
| 9 | video.play() blocking getUserMedia | Fixed | Fire-and-forget, loop gates on readyState |
| 10 | AnchorContent wrong scale for back camera | Fixed (untested) | flip={camera==='front'} |
| 11 | Overlay jitter / wrong direction | In progress | AnchorContent fix + filter tuning |
| 12 | Tracking infrequent on e-cell card | In progress | Low-texture card; CLAHE shader next |

---

## 10. Session Log

| Date | What happened |
|---|---|
| 2026-05-20 | graphify run — 309 nodes, 435 edges knowledge graph built |
| 2026-05-20 | Fixed onReady loop, AnchorContent flip, texture.flipY, debug panel |
| 2026-05-21 | WebGL preprocessing pipeline built + debugged (infinite loop, Y-flip, 30fps cap) |
| 2026-05-21 | Glare toggle fixed via GLSL bypass uniform |
| 2026-05-21 | AnchorContent flip corrected for back vs front camera |
| 2026-05-21 | Major debug panel rewrite: FPS, profile presets, event counters, reset button |
| 2026-05-21 | This document written |

---

*Keep updating this document. Every session should add at least one line to §10 and update any status changes in §3 and §9.*
