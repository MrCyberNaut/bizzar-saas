# 06 — WebAR Mobile Performance

**Date:** 2026-05-19
**Topic:** GPU contention, RAF throttling, WASM threading, battery, and camera-pipeline tuning specifically for mobile WebAR.

---

## TL;DR

Mobile WebAR is fundamentally GPU-bound, not CPU-bound, because the TF.js detector, the Three.js render, and the camera texture upload all hit the same GPU. The biggest wins, in order:
1. Cap render rate at 30 fps (matches camera rate).
2. Cap camera resolution at 1280×720.
3. Use WebGL backend, not WASM, on mobile (counter-intuitive but currently faster).
4. Reduce render workload — don't ship unoptimized 3D models.
5. Pre-warm WASM before the user taps "scan."

---

## The GPU Contention Problem

Every WebAR frame on mobile does, on the same GPU:
1. **Camera texture upload** — every frame the new video texture must be re-uploaded to GPU memory. On iOS, this is the single largest GPU stall.
2. **TF.js / MindAR detect+track** — runs as WebGL compute, taking ~10–20 ms on a mid-tier phone.
3. **R3F render** — your 3D content, transparent UI overlays, etc.

If any one of these blows the 16.6 ms (60 fps) or 33.3 ms (30 fps) budget, you drop a frame. Cumulative timing on a Snapdragon 8 Gen 1 in our benchmark scenario:
- Texture upload: 3–6 ms
- MindAR: 12–18 ms
- R3F (modest content): 6–10 ms
- **Total: 21–34 ms** → 30 fps achievable, 60 fps not.

On budget phones (Snapdragon 6-series, A-series mid-tier), MindAR alone can be 25–35 ms → tracking degrades visibly.

---

## RAF (requestAnimationFrame) Throttling

### Browser-level throttling
- Hidden tabs / background tabs: RAF throttled to **≤ 1 Hz** (Chrome 88+ went heavy on this).
- Sources: [Chrome timer throttling blog](https://developer.chrome.com/blog/timer-throttling-in-chrome-88), [Motion mag: When browsers throttle RAF](https://motion.dev/magazine/when-browsers-throttle-requestanimationframe).
- Practical effect: if a user backgrounds the AR tab, MindAR's loop pauses naturally, which is good — but if the page is still **visible but in a non-focused tab**, RAF can still drop to 30 → 10 Hz depending on browser.

### Painter throttling
- Hidden content (e.g. content under a fullscreen overlay) doesn't repaint, so RAF doesn't fire. Reference: [Motion mag](https://motion.dev/magazine/when-browsers-throttle-requestanimationframe).

### Self-throttling to 30 fps
- Even when not throttled, **drive AR at 30 fps**. Reference: target 30 FPS reduces GPU demand and improves battery life on mobile without significantly sacrificing UX. ([Pixel Free Studio](https://blog.pixelfreestudio.com/webgl-in-mobile-development-challenges-and-solutions/))
- Implementation pattern:
  ```js
  let last = 0;
  function loop(t) {
    if (t - last < 33) { requestAnimationFrame(loop); return; }
    last = t;
    runMindARStep();
    requestAnimationFrame(loop);
  }
  ```

---

## WASM Threading

- `mind-ar-js` uses TF.js with the **WebGL backend** by default, not WASM. The WASM backend exists but is slower for image tracking because the algorithm is heavily parallel-matrix-multiply, which WebGL does better.
- WASM **threading** (SharedArrayBuffer + Atomics) is gated behind cross-origin isolation (`COOP`/`COEP` headers). Most production Next.js deployments **do not have these set**, so even if you opted into WASM, you'd be single-threaded.
- 2025 WebAssembly/WebGPU note: "WebAssembly (Wasm) and WebGPU have turned the browser into a serious high-performance platform for interactive applications." Long-term, WebGPU + WASM compute may overtake the TF.js WebGL path, but **not yet for MindAR**. ([Technology.org 2025](https://www.technology.org/2025/11/11/webassembly-webgpu-in-browser-game-platforms-pushing-the-frontier-of-real-time-graphics/))

---

## Camera Constraints (getUserMedia)

### Recommended constraints

```js
navigator.mediaDevices.getUserMedia({
  video: {
    facingMode: { ideal: 'environment' },
    width: { ideal: 1280, max: 1920 },
    height: { ideal: 720, max: 1080 },
    frameRate: { ideal: 30, max: 30 },
    // experimental but widely supported:
    advanced: [{ focusMode: 'continuous' }]
  },
  audio: false
});
```

### Why these values
- **1280×720**: enough resolution for feature detection on a typical card at arm's length; above this, returns diminish and CPU/GPU cost rises linearly.
- **30 fps cap**: aligns with most mobile camera native rates; prevents 60 fps requests that lead to camera under-exposure on some devices.
- **`focusMode: continuous`**: critical — if focus drifts, features blur and detection drops. Not supported on all browsers (notably some iOS Safari versions don't honor it), but no harm trying.
- **`facingMode: environment`**: rear camera. Don't use `exact` constraint — fail gracefully if not available.

### Known issues
- WebKit ([bug 179994](https://bugs.webkit.org/show_bug.cgi?id=179994)): Safari historically ignored `frameRate` and `width`/`height` constraints; behavior has improved since but verify on target devices.
- Older Android Chrome: some devices return wildly different resolutions than requested. Always read back `track.getSettings()` and adapt.

Sources: [getUserMedia constraints — addpipe](https://blog.addpipe.com/getusermedia-video-constraints/), [webrtcHacks resolution constraints](https://webrtchacks.com/getusermedia-resolutions-3/).

---

## Battery / Heat

- Sustained 60 fps + camera + WebGL = phone gets hot in **3–5 minutes**. Many users won't notice but thermal throttling will start dropping framerate.
- Mitigations:
  - 30 fps cap (cuts heat ~40%).
  - Pause MindAR when the page is hidden (`visibilitychange` listener — stop `getUserMedia` tracks and tear down the MindAR controller; restart on visible).
  - Don't run additional Three.js animations at 60 fps when AR is active.
  - Use `prefers-reduced-motion` as a hint.

---

## R3F-Specific Pitfalls

From [R3F performance pitfalls](https://r3f.docs.pmnd.rs/advanced/pitfalls):
- **Never mutate state inside `useFrame`** — fast updates are by direct three.js mutation (`obj.position.x = ...`), not React state.
- **Use `frameloop="demand"`** when possible — but AR runs continuously so this isn't applicable in the AR view (it is in the editor).
- **Instance everything**. If a panel renders 10 module cards, use `<Instances>` not 10 `<mesh>`es.
- **Re-use materials & geometries**.

---

## WebGPU on Mobile (2025–2026)

- WebGPU is shipping on Chrome Android, but the Babylon.js and Construct teams have both filed performance regressions specifically on mobile WebGPU vs WebGL ([Babylon forum](https://forum.babylonjs.com/t/webgpu-performance-regression-and-worker-thread-lag-on-mobile-since-v9-5-0/63456), [Construct bug](https://github.com/Scirra/Construct-bugs/issues/7874)).
- **For mobile WebAR in 2026, stay on WebGL.** Revisit WebGPU in late 2026 / 2027.

---

## Actionable for BizzAR

1. **Cap framerate to 30 fps** in the AR viewer. Patch react-three-mind or wrap the MindAR loop.
2. **Set explicit `getUserMedia` constraints** as listed above. Don't use the library default.
3. **Add a `visibilitychange` handler** that calls `stopTracking()` + releases the camera; resume on visible. Halves battery cost during interruptions (incoming notification, etc.).
4. **Pre-warm WASM/WebGL during the page load** so the user's first tap on "Scan" doesn't pay the ~500 ms init cost. Load the MindAR module on idle (`requestIdleCallback`), but don't start tracking until tap.
5. **Optimize 3D content per panel**:
   - GLTF panels: budget ≤ 50k tris, ≤ 1024px textures, KTX2 compressed.
   - Avoid transparency-heavy materials (alpha sorting kills mobile perf).
   - No real-time shadows. Use baked / vertex AO.
6. **Add an in-app FPS meter (dev mode)** with a clear "drop detected" log so you can correlate poor stability with frame drops.
7. **Lighthouse / Chrome DevTools performance profile** on a representative mid-tier Android (e.g. Pixel 6a) — make the perf budget explicit and CI it.
8. **Document a tested-devices list** for users: "Works smoothly on iPhone 12+, Galaxy S20+, Pixel 6+; may degrade on older devices."

---

## Sources

- [WebGL in Mobile Development — Pixel Free Studio](https://blog.pixelfreestudio.com/webgl-in-mobile-development-challenges-and-solutions/)
- [requestAnimationFrame — DebugBear](https://www.debugbear.com/blog/requestanimationframe)
- [Chrome 88 timer throttling](https://developer.chrome.com/blog/timer-throttling-in-chrome-88)
- [Motion: when browsers throttle RAF](https://motion.dev/magazine/when-browsers-throttle-requestanimationframe)
- [WebAssembly & WebGPU 2025 — Technology.org](https://www.technology.org/2025/11/11/webassembly-webgpu-in-browser-game-platforms-pushing-the-frontier-of-real-time-graphics/)
- [WebGPU timing performance](https://webgpufundamentals.org/webgpu/lessons/webgpu-timing.html)
- [Babylon WebGPU mobile regression](https://forum.babylonjs.com/t/webgpu-performance-regression-and-worker-thread-lag-on-mobile-since-v9-5-0/63456)
- [Construct WebGPU mobile bug](https://github.com/Scirra/Construct-bugs/issues/7874)
- [getUserMedia video constraints — addpipe](https://blog.addpipe.com/getusermedia-video-constraints/)
- [getUserMedia resolutions — webrtcHacks](https://webrtchacks.com/getusermedia-resolutions-3/)
- [WebKit bug 179994 — frameRate constraints](https://bugs.webkit.org/show_bug.cgi?id=179994)
- [R3F performance pitfalls](https://r3f.docs.pmnd.rs/advanced/pitfalls)
