# BizzAR — Error Log

Format per entry:
## [DATE] — [Short title]
- **Error:** exact error message or description
- **Context:** what we were doing
- **Status:** open | fixed
- **Fix:** (fill in when resolved)

---

## 2026-04-24 — 8th Wall replaced with MindAR.js
- **Error:** 8th Wall cloud SDK is paid (free tier removed by Niantic). `NEXT_PUBLIC_8THWALL_KEY` required, no free access.
- **Context:** Trying to run bizzar-proto AR engine
- **Status:** fixed
- **Fix:** Replaced `XR8` / `XRExtras` with MindAR.js (free, open source). Rewrote `ARScene.tsx`, `ARPageClient.tsx`, `page.tsx`. No API key needed.

## 2026-04-24 — MindAR .mind compiler CLI fails on Node 24
- **Error:** `node-pre-gyp ERR! not ok` — native bindings fail to compile on Node v24.14.1
- **Context:** Trying to compile card1.jpg → card.mind via `npm install hiukim/mind-ar-js`
- **Status:** fixed
- **Fix:** Built headless multi-target compiler script `scripts/compile-mind.js` using Playwright Chromium to drive the browser compiler automated without native Node bindings. Multi-target compilation for card1.jpg, card2.jpg, card3.png runs headlessly via `node scripts/compile-mind.js`.

## 2026-04-08 — drei Html distanceFactor bug with orthographic camera
- **Error:** Module labels filled entire viewport — Html elements scaled to enormous size
- **Context:** 3D editor (hub-scene) with orthographic camera + drei Html transform mode
- **Status:** fixed
- **Fix:** Removed `distanceFactor` from drei `Html` component. With orthographic cameras, distanceFactor causes runaway scaling. Use `distanceFactor={1.5}` max or omit entirely.

## 2026-04-08 — drei Html overlay blocks R3F raycasting (panel click broken)
- **Error:** Clicking panels in 3D editor did not fire R3F onClick — clicks intercepted by Html DOM layer
- **Context:** Panel selection in hub-scene.tsx
- **Status:** fixed
- **Fix:** Added `eventSource={containerRef}` + `eventPrefix="client"` to Canvas. Added `wrapperClass="panel-html-overlay"` to Html. Added CSS `pointer-events: none` on `.panel-html-overlay *`.

## 2026-04-08 — Supabase project auto-paused (free tier)
- **Error:** DNS resolution fails for `wyujyqrblovrmnymfmms.supabase.co` — project INACTIVE
- **Context:** Dashboard shows 0 cards, all DB queries silently return null
- **Status:** fixed (recurring)
- **Fix:** Restore via Supabase MCP tool: `restore_project("wyujyqrblovrmnymfmms")`. Wait ~3 min for ACTIVE_HEALTHY. Will recur on free tier after inactivity.

## 2026-05-06 — Camera not launching in AR scene (wrong MindAR bundle)
- **Error:** Black screen, camera never prompts. `window.MINDAR.IMAGE.MindARThree` undefined.
- **Context:** bizzar-proto `/ar` page on phone via ngrok
- **Status:** fixed
- **Fix:** Changed CDN script from `mindar-image.prod.js` (compiler only, no Three.js) to `mindar-image-three.prod.js` (includes MindARThree class). Also rewrote ARScene.tsx to sync MindAR's camera projection matrix to R3F camera via `CameraSync` component using `useFrame`.

## 2026-05-07 — Three.js version conflict between MindAR and @react-three/drei
- **Error:** `BatchedMesh` not exported from 'three' — drei v9.105 requires three >= 0.155. `sRGBEncoding` not exported — MindAR 1.2.5 requires three <= 0.151.
- **Context:** bizzar-proto after installing mind-ar npm package
- **Status:** fixed
- **Fix:** Downgraded `three` to `0.150.1` and `@react-three/drei` to `9.88.0` in bizzar-proto (proto is isolated — doesn't affect main bizzar app). Residual warning "Multiple instances of Three.js" is harmless — MindAR bundles its own Three.js internally.

## 2026-05-14 — MindARThree is not a constructor — FIXED
- **Error:** `MindARThree is not a constructor` on phone AR load
- **Root cause:** `mind-ar/dist/mindar-image-three.prod.js` is a webpack IIFE bundle that assigns to `window.MINDAR.IMAGE.MindARThree` — it has NO ES module exports. Destructuring `{ MindARThree }` from a dynamic import always yields `undefined`.
- **Fix:** Changed dynamic import to: `await import('...'); const MindARThree = (window as any).MINDAR?.IMAGE?.MindARThree`
- **Files changed:** `src/components/ar/ARScene.tsx`, `src/types/mind-ar.d.ts`
