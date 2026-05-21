# @react-three/drei — AR Patterns

**Date:** 2026-05-17
**Relevance:** Every drei component used in BizzAR panels

---

## 1. `<Html>` Component — AR Caveats

### transform mode vs screen-space
```typescript
// transform mode — HTML element follows 3D position WITH perspective scaling
// Required for panels that should "float" in 3D space
<Html transform occlude={false} position={[0, 0.8, 0.1]}>
  <div style={{ width: 180 }}>...</div>
</Html>

// screen-space (no transform prop) — HTML element at fixed screen position
// 3D position just determines WHERE on screen, not scale
<Html position={[0, 0.8, 0.1]}>
  <div>Always same size regardless of distance</div>
</Html>
```

### distanceFactor — DO NOT USE with orthographic cameras
From CLAUDE.md: "Do NOT use `distanceFactor` on drei `Html` with orthographic cameras — causes massive scaling."

With perspective camera (react-three-mind uses perspective) — distanceFactor works:
```typescript
// Scales element inversely with distance — further = smaller
<Html transform distanceFactor={10} position={[0, 0.8, 0.1]}>
```

But for BizzAR: we've moved to native R3F meshes — `<Html>` is no longer used for panels. Avoid entirely.

### eventSource + eventPrefix (if Html ever comes back)
```typescript
// Canvas must forward events to Html elements
<Canvas eventSource={document.getElementById('root')!} eventPrefix="client">
  <Html transform>
    <button onClick={...}>clickable</button>
  </Html>
</Canvas>
```

Without `eventSource`, Html pointer events block R3F raycasting.

---

## 2. `<Text>` — SDF Font Rendering

```typescript
import { Text } from '@react-three/drei'

<Text
  position={[0, 0, 0.01]}
  fontSize={0.035}          // world units
  color="#ffffff"
  anchorX="center"
  anchorY="middle"
  maxWidth={0.4}            // word wrap at 0.4 world units
  font="/fonts/Inter-Bold.woff"  // optional custom font
>
  + Add to Contacts
</Text>
```

**Performance:** Troika-three-text renders SDF (Signed Distance Field) in a Web Worker — doesn't block main thread. First render has ~50ms delay while font loads. After that, GPU-accelerated at any scale.

**Mobile:** SDF is much better than canvas-texture text for sharp rendering at any DPR.

---

## 3. `useTexture` — Caching and Preloading

```typescript
import { useTexture } from '@react-three/drei'

// Inside Suspense — throws promise until loaded
function ContactCard() {
  const texture = useTexture('/card1.jpg')
  // texture is a THREE.Texture, ready to use
}

// Preload outside component tree (avoids Suspense delay)
useTexture.preload('/card1.jpg')

// Multiple textures
const [cardTex, logoTex] = useTexture(['/card1.jpg', '/logo.png'])
```

**Cache:** drei caches by URL — calling `useTexture('/card1.jpg')` in multiple components reuses the same texture.

---

## 4. `useGLTF` — 3D Models

```typescript
import { useGLTF } from '@react-three/drei'

function Model3D() {
  const { scene } = useGLTF('/model.glb')
  return <primitive object={scene.clone()} />
}

// Preload
useGLTF.preload('/model.glb')

// With Draco compression — requires CDN decoder
useGLTF.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/')
```

---

## 5. `<Billboard>` — Always Faces Camera

Useful for label panels that should always face the user regardless of card orientation:

```typescript
import { Billboard } from '@react-three/drei'

// Rotates around Y-axis only (most natural for floating labels)
<Billboard follow={true} lockX={false} lockY={true} lockZ={false}>
  <Text fontSize={0.04}>Arnav Sawant</Text>
</Billboard>
```

**BizzAR use case:** Name/title label that always faces camera even as user tilts the card.

---

## 6. `<RoundedBox>` — Buttons

```typescript
import { RoundedBox } from '@react-three/drei'

<RoundedBox
  args={[0.44, 0.1, 0.008]}  // width, height, depth
  radius={0.012}               // corner radius
  smoothness={4}               // segments for rounded corners
>
  <meshBasicMaterial color="#6366f1" />
</RoundedBox>
```

Currently used for the "Add to Contacts" button in ARScene.tsx ✅

---

## 7. `<Sparkles>` — Particle Effect on Card Detection

```typescript
import { Sparkles } from '@react-three/drei'

// Show sparkles when card is first detected
{isTracking && (
  <Sparkles
    count={30}
    scale={[1.2, 0.8, 0.1]}   // area around card
    size={0.08}
    speed={0.3}
    color="#6366f1"
  />
)}
```

GPU-instanced particles — cheap on mobile. Great "wow moment" on first scan.

---

## 8. `<Preload>` — Warm Up GPU

```typescript
import { Preload } from '@react-three/drei'

// Inside Canvas/ARView — triggers first-frame compilation immediately
// Prevents jank on first visible render
<ARView ...>
  <ARContent />
  <Preload all />
</ARView>
```

---

## 9. `<Decal>` — Surface Overlays

For overlaying a logo/QR onto the card mesh:

```typescript
import { Decal, useTexture } from '@react-three/drei'

function CardWithDecal() {
  const decalTex = useTexture('/logo.png')
  return (
    <mesh>
      <planeGeometry args={[1, 0.635]} />
      <meshBasicMaterial />
      <Decal
        position={[0.3, 0, 0.001]}
        rotation={[0, 0, 0]}
        scale={[0.2, 0.2, 0.2]}
        map={decalTex}
      />
    </mesh>
  )
}
```

---

## Actionable for BizzAR

1. **Now:** Add `<Sparkles>` on `onAnchorFound` for card detection "pop" effect — ~5 lines
2. **Now:** Add `<Preload all />` inside ARView — eliminates first-frame jank
3. **Now:** `useTexture.preload('/card1.jpg')` at module level — texture ready before card detected
4. **Later:** `<Billboard>` for name label that always faces camera — better than fixed rotation
5. **Later:** Preload 3D model GLB on idle

---

## Sources
- https://drei.docs.pmnd.rs/misc/html
- https://drei.docs.pmnd.rs/abstractions/text
- https://drei.docs.pmnd.rs/loaders/use-texture
- https://drei.docs.pmnd.rs/shapes/rounded-box
- https://drei.docs.pmnd.rs/staging/sparkles
- https://github.com/pmndrs/drei
