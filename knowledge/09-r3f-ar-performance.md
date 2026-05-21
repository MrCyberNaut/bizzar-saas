# React Three Fiber — AR Performance Deep Dive

**Date:** 2026-05-17
**Relevance:** Every render decision in bizzar-proto

---

## 1. frameloop — Always Use "always" for AR

```typescript
// react-three-mind ARView passes through to R3F Canvas
// MindAR mutates Three.js objects directly every frame via matrixWorld
// "demand" goes stale — tracking updates won't trigger re-renders
// "always" is mandatory for AR
```

`frameloop="demand"` would only re-render when React state changes — but MindAR updates `anchor.group.matrixWorld` imperatively outside React. The canvas would freeze with stale transforms. **Never use `"demand"` with MindAR.**

---

## 2. Production gl Config

```typescript
<ARView
  // Passed through to R3F Canvas gl prop
  gl={{
    antialias: false,           // MSAA is expensive on mobile — skip for AR overlays
    precision: 'mediump',       // mediump vs highp: ~10% faster on Adreno, imperceptible quality diff
    stencil: false,             // not needed for AR overlays
    alpha: true,                // required — canvas must be transparent over camera feed
    powerPreference: 'high-performance',
  }}
  dpr={[1, 1.5]}               // cap device pixel ratio — biggest single perf win
>
```

---

## 3. useFrame Priority System

```typescript
// useFrame(callback, priority)
// Lower number = runs first. Default = 0.
// MindAR updates happen at priority -1 internally.

// Run AFTER MindAR updates — priority 1+
useFrame(({ clock }, delta) => {
  if (!ref.current) return
  // delta-time animation — frame-rate independent
  ref.current.rotation.y += delta * 0.5
}, 1)
```

**Avoid per-frame allocations:**
```typescript
// ❌ Allocates new Vector3 every frame
useFrame(() => {
  mesh.position.copy(new THREE.Vector3(x, y, z))
})

// ✅ Mutate existing object
const targetPos = useRef(new THREE.Vector3())
useFrame(() => {
  targetPos.current.set(x, y, z)
  mesh.position.lerp(targetPos.current, 0.1)
})
```

---

## 4. react-three-mind ARView → Canvas Prop Passthrough

All standard R3F Canvas props work on ARView:

```typescript
<ARView
  // MindAR props
  imageTargets="/card.mind"
  filterMinCF={0.0001}
  filterBeta={5000}
  missTolerance={25}
  warmupTolerance={2}
  maxTrack={1}

  // R3F Canvas props (passed through via ...s spread in react-three-mind source)
  dpr={[1, 1.5]}
  gl={{ antialias: false, alpha: true, powerPreference: 'high-performance' }}
  style={{ position: 'fixed', inset: 0, zIndex: 0 }}

  // Callbacks
  onReady={() => setReady(true)}
  onError={(e) => console.warn(e)}
>
```

---

## 5. Geometry and Material Hoisting

```typescript
// ❌ New geometry created on every render
function Panel() {
  return (
    <mesh>
      <planeGeometry args={[0.7, 0.44]} />  {/* recreated each render */}
```

```typescript
// ✅ Hoist outside component — created once
const PANEL_GEO = new THREE.PlaneGeometry(0.7, 0.44)

function Panel() {
  return (
    <mesh geometry={PANEL_GEO}>
```

For BizzAR panels (static geometry), this is a meaningful optimization.

---

## 6. Suspense Placement

```typescript
// ❌ Suspense at ARContent root — blanks entire canvas while loading
<ARAnchor target={0}>
  <Suspense fallback={null}>
    <CardFrame />
    <ContactCard />  {/* useTexture throws promise here */}
  </Suspense>
</ARAnchor>

// ✅ Suspense per-module — CardFrame renders immediately, ContactCard loads async
<ARAnchor target={0}>
  <CardFrame />
  <Suspense fallback={null}>
    <ContactCard />
  </Suspense>
</ARAnchor>
```

---

## 7. Profiling Tools

```bash
npm install --save-dev r3f-perf
```

```typescript
import { Perf } from 'r3f-perf'

// Inside Canvas/ARView children — dev only
{process.env.NODE_ENV === 'development' && <Perf position="top-left" />}
```

Shows: FPS, GPU ms, CPU ms, draw calls, triangle count. Essential for AR optimization.

---

## 8. Instanced Mesh (for future multi-panel)

When adding multiple panels back (socials, GitHub, gallery), use instancing if they share geometry:

```typescript
// Up to 1000 instances at cost of one draw call
<instancedMesh ref={ref} args={[undefined, undefined, count]}>
  <planeGeometry args={[0.5, 0.3]} />
  <meshBasicMaterial />
</instancedMesh>
```

---

## Actionable for BizzAR

1. **Now:** Add `gl={{ antialias: false, alpha: true, powerPreference: 'high-performance' }}` to ARView
2. **Now:** Add `<Perf />` in dev mode — see actual GPU/CPU cost
3. **Now:** Hoist `planeGeometry` constants outside components
4. **Now:** Per-module Suspense (already done for ContactCard ✅)
5. **Later:** Instanced mesh when adding multiple panels back

---

## Sources
- https://docs.pmnd.rs/react-three-fiber/advanced/performance
- https://github.com/pmndrs/react-three-fiber/blob/master/docs/advanced/performance.mdx
- https://github.com/RenaudRohlinger/r3f-perf
- https://docs.pmnd.rs/react-three-fiber/api/canvas (Canvas props)
