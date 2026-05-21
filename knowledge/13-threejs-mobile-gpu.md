# Three.js Mobile GPU Performance

**Date:** 2026-05-17
**Relevance:** AR rendering budget, frame rate targets, battery life on mid-range Android + iPhone

---

## 1. WebGL1 vs WebGL2

**Target WebGL2 with graceful fallback:**
```typescript
// R3F — WebGL2 is default in Three.js r150+
<Canvas gl={{ powerPreference: 'high-performance' }}>
```
- iOS 15+ / Android Chrome 58+ / Firefox 51+: WebGL2 ✅
- Older devices fall back to WebGL1 automatically via Three.js renderer
- **AR benefit:** WebGL2 enables instanced rendering + better texture formats

---

## 2. Draw Call Budget

| GPU Family | Max draw calls/frame (60fps) |
|---|---|
| Apple A14+ | ~300–500 |
| Apple A12 (iPhone XR era) | ~200 |
| Adreno 6xx (Pixel 5-7) | ~80–150 |
| Adreno 5xx (Pixel 4 era) | ~50–80 |
| Mali G-series (Samsung mid-range) | ~80–150 |

**BizzAR AR overlay target: ≤ 30 draw calls**

Current scene (1 CardFrame + 1 photo plane + 1 button + 1 text):
- ~4–6 draw calls. Well within budget. ✅

---

## 3. Material Cost Ladder

```
MeshBasicMaterial     — 0 lighting calcs, cheapest
MeshLambertMaterial   — diffuse only, cheap
MeshPhongMaterial     — specular, moderate
MeshStandardMaterial  — PBR, expensive on mobile
MeshPhysicalMaterial  — full PBR, very expensive
```

**Rule for BizzAR:**
- All 2D panels / contact card / buttons → `MeshBasicMaterial`
- 3D models only → `MeshStandardMaterial` (module-level opt-in)

```typescript
// ✅ Correct — panels
<meshBasicMaterial map={texture} toneMapped={false} />

// ❌ Avoid for flat panels
<meshStandardMaterial map={texture} />
```

---

## 4. Texture Compression (KTX2 / Basis Universal)

Reduces GPU memory 4–6× vs PNG/JPEG for runtime textures:

```bash
# CLI: convert card image to KTX2 with Basis Universal
npx @gltf-transform/cli ktx2 card1.jpg card1.ktx2 --slots baseColor
```

```typescript
// R3F with drei
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader'
import { useLoader } from '@react-three/fiber'

// Or use drei's useTexture which handles KTX2 via file extension
const texture = useTexture('/card1.ktx2')
```

**Caveat:** KTX2 transcode requires WASM + SharedArrayBuffer. Need COOP/COEP headers (see section 7).
For v1, JPEG textures are fine — card image is one texture, GPU memory is not the bottleneck.

---

## 5. Transparency Best Practices

```typescript
// Icon cutouts (sharp edges) — use alphaTest, avoids sort cost
<meshBasicMaterial map={iconTex} alphaTest={0.5} transparent={false} />

// Overlaid glass panels — depthWrite: false prevents z-fighting
<meshBasicMaterial color="#000" transparent opacity={0.85} depthWrite={false} />

// Avoid fully alpha-blended layered transparent objects — GPU must sort every frame
```

---

## 6. DPR Cap + Antialias

**Single biggest mobile perf lever:**
```typescript
// In ARView (react-three-mind passes to Canvas)
dpr={[1, 1.5]}  // ← current setting ✅
```

Why: Pixel 7 Pro has 3× DPR. Without cap: 3× = 9× more pixels rendered vs 1×.
At 1.5 DPR cap: renders at 2.25× pixels — good enough quality, ~60% faster than 3×.

**Antialias:**
```typescript
gl={{ antialias: false }}  // Default in react-three-mind? Verify.
```
MSAA on mobile is expensive. For AR (camera feed as background), slight aliasing on 3D overlays is acceptable.

---

## 7. WASM + SharedArrayBuffer (MindAR threading)

MindAR uses TF.js WebGL backend + Web Worker. For WASM threading (SharedArrayBuffer), browser requires:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**Next.js config — scoped to /ar route only** (global breaks Stripe, OAuth, iframes):
```javascript
// next.config.js
const nextConfig = {
  async headers() {
    return [
      {
        source: '/ar/:path*',
        headers: [
          { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
          { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
        ],
      },
    ]
  },
}
```

**Note:** MindAR v1.2.5 doesn't publish a multi-threaded WASM build. TF.js WebGL backend runs on GPU anyway — COOP/COEP is future-proofing for when it does.

---

## 8. rAF Throttling on Mobile

- **Background tab:** Browser stops calling rAF entirely. Tracking pauses.
- **iOS Low Power Mode:** rAF throttled to 30fps even when visible
- **Android battery saver:** Similar throttling

```typescript
// Handle visibility changes in ARScene
useEffect(() => {
  const onVisibility = () => {
    if (document.hidden) {
      // Optional: pause tracking to save battery
    }
  }
  document.addEventListener('visibilitychange', onVisibility)
  return () => document.removeEventListener('visibilitychange', onVisibility)
}, [])
```

---

## 9. Memory Management

```typescript
// Dispose geometry and material when component unmounts
useEffect(() => {
  return () => {
    geometry.dispose()
    material.dispose()
    texture.dispose()
  }
}, [])

// Debug: check renderer memory usage
const { gl } = useThree()
console.log(gl.info.memory) // { geometries, textures }
console.log(gl.info.render)  // { calls, triangles, points, lines }
```

---

## Actionable for BizzAR

1. **Done:** `dpr={[1, 1.5]}` — biggest perf win already in place
2. **Done:** All panels use `MeshBasicMaterial` — correct
3. **Add:** COOP/COEP headers scoped to `/ar/*` in next.config.js (future WASM threading)
4. **Add:** `visibilitychange` handler to pause/resume tracking — cuts battery drain
5. **Add:** `gl={{ antialias: false, powerPreference: 'high-performance' }}` to ARView
6. **Later:** KTX2 textures for user-uploaded card photos (after v1 ships)

---

## Sources
- https://threejs.org/docs/#api/en/renderers/WebGLRenderer
- https://github.com/pmndrs/react-three-fiber/discussions (performance threads)
- https://web.dev/gpu-compute/ (GPU optimization)
- https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/WebGL_best_practices
- https://gltf.report (GLTF optimization tool)
- https://github.com/nicktindall/cycl (SharedArrayBuffer requirements)
