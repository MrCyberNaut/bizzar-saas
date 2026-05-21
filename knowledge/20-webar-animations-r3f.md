# WebAR Animations in R3F

**Date:** 2026-05-17
**Relevance:** Panel entrance animations, particle effects, 3D model animations in BizzAR

---

## 1. @react-spring/three — Spring Physics Animations

Best for panel pop-in / scale animations tied to tracking state:

```bash
npm install @react-spring/three
```

```typescript
import { useSpring, animated } from '@react-spring/three'

function ContactCard({ visible }: { visible: boolean }) {
  const { scale, opacity } = useSpring({
    scale: visible ? 1 : 0,
    opacity: visible ? 1 : 0,
    config: { tension: 280, friction: 22 },  // springy pop
    delay: visible ? 100 : 0,               // slight delay after card detected
  })

  return (
    // animated.group is the R3F-compatible animated component
    <animated.group scale={scale} position={[0, 0.82, 0.05]}>
      <mesh>
        <planeGeometry args={[0.7, 0.44]} />
        <animated.meshBasicMaterial map={texture} transparent opacity={opacity} />
      </mesh>
    </animated.group>
  )
}
```

**Config presets:**
```typescript
import { config } from '@react-spring/three'

config.wobbly    // { tension: 180, friction: 12 } — bouncy
config.stiff     // { tension: 210, friction: 20 } — snappy
config.gentle    // { tension: 120, friction: 14 } — smooth
```

---

## 2. useFrame Lerp/Slerp — Smooth Tracking Follow

For smoothing position/rotation of content as the card moves:

```typescript
import { useRef } from 'react'
import { useFrame } from '@react-three/fiber'
import * as THREE from 'three'

function SmoothPanel({ targetPosition }: { targetPosition: THREE.Vector3 }) {
  const groupRef = useRef<THREE.Group>(null)
  const currentPos = useRef(new THREE.Vector3())

  useFrame((_, delta) => {
    if (!groupRef.current) return
    // Lerp toward target — alpha based on delta for frame-rate independence
    const alpha = 1 - Math.exp(-15 * delta)  // ~15 = speed factor
    currentPos.current.lerp(targetPosition, alpha)
    groupRef.current.position.copy(currentPos.current)
  })

  return <group ref={groupRef}>...</group>
}

// Quaternion slerp for rotation smoothing
const currentQuat = useRef(new THREE.Quaternion())
useFrame((_, delta) => {
  const alpha = 1 - Math.exp(-15 * delta)
  currentQuat.current.slerp(targetQuat, alpha)
  groupRef.current.quaternion.copy(currentQuat.current)
})
```

**Note:** MindAR + react-three-mind already smooth the pose via One Euro Filter. Additional lerp adds latency — only use for extra smoothness on panel entrance, not for tracking follow.

---

## 3. GSAP with R3F

```bash
npm install gsap
```

```typescript
import { gsap } from 'gsap'
import { useEffect, useRef } from 'react'
import * as THREE from 'three'

function AnimatedPanel() {
  const groupRef = useRef<THREE.Group>(null)

  useEffect(() => {
    if (!groupRef.current) return
    // Entrance animation — scale from 0 to 1
    gsap.fromTo(
      groupRef.current.scale,
      { x: 0, y: 0, z: 0 },
      { x: 1, y: 1, z: 1, duration: 0.4, ease: 'back.out(1.7)' }
    )
  }, [])

  return <group ref={groupRef}>...</group>
}
```

GSAP timeline for sequenced panel reveals:
```typescript
const tl = gsap.timeline()
tl.fromTo(cardFrameRef.current!.scale, { x: 0, y: 0, z: 0 }, { x: 1, y: 1, z: 1, duration: 0.3 })
  .fromTo(contactRef.current!.scale,   { x: 0, y: 0, z: 0 }, { x: 1, y: 1, z: 1, duration: 0.4, ease: 'back.out(1.7)' }, '+=0.1')
```

---

## 4. Framer Motion 3D

```bash
# Note: @framer-motion/three was deprecated for React 19
# Use motion from 'framer-motion' with R3F via motion.create() in latest
npm install framer-motion
```

For BizzAR, `@react-spring/three` is the better choice — more stable, purpose-built for Three.js, and actively maintained.

---

## 5. `<Sparkles>` — Ready-Made Particles

```typescript
import { Sparkles } from '@react-three/drei'

// On card detection
<ARAnchor target={0} onAnchorFound={onFound} onAnchorLost={onLost}>
  {isTracking && (
    <Sparkles
      count={35}
      scale={[1.1, 0.7, 0.08]}   // bounding box — cover the card
      size={0.07}                  // particle size in world units
      speed={0.5}                  // animation speed
      color="#818cf8"              // indigo to match BizzAR brand
      opacity={0.8}
      noise={0.2}
    />
  )}
```

`<Sparkles>` uses GPU instancing — all 35 particles = 1 draw call. ✅

---

## 6. GPU vs CPU Particles on Mobile

| Approach | Draw Calls | CPU Cost | GPU Cost | Mobile Performance |
|---|---|---|---|---|
| `<Sparkles>` (instanced) | 1 | Low | Low | ✅ Excellent |
| Manual instanced mesh | 1 | Low | Low | ✅ Excellent |
| Individual mesh per particle | N | Low | High | ❌ Avoid |
| CSS/DOM animation | 0 (WebGL) | Medium | 0 (GPU) | ✅ Good for HUD |

For AR overlays: always use instanced geometry for particle effects.

---

## 7. GLTF Skeleton Animations (useAnimations)

For the 3D model module — when a GLB has embedded animations:

```typescript
import { useGLTF, useAnimations } from '@react-three/drei'

function AnimatedModel() {
  const groupRef = useRef<THREE.Group>(null)
  const { scene, animations } = useGLTF('/model.glb')
  const { actions } = useAnimations(animations, groupRef)

  useEffect(() => {
    // Play the first animation on mount
    const action = actions[Object.keys(actions)[0]]
    action?.reset().play()
  }, [actions])

  return <group ref={groupRef}>
    <primitive object={scene} />
  </group>
}
```

---

## 8. Panel Entrance Sequence (Recommended Pattern)

Stagger panel reveals for a polished AR experience:

```typescript
function ARContent({ isTracking }: { isTracking: boolean }) {
  // Each panel has a delay — creates a reveal sequence
  const delays = { frame: 0, contact: 100, socials: 250, github: 400 }

  return (
    <ARAnchor target={0} ...>
      <AnimatedPanel visible={isTracking} delay={delays.frame}>
        <CardFrame />
      </AnimatedPanel>
      <AnimatedPanel visible={isTracking} delay={delays.contact}>
        <ContactCard />
      </AnimatedPanel>
    </ARAnchor>
  )
}

function AnimatedPanel({ visible, delay, children }) {
  const { scale } = useSpring({
    scale: visible ? 1 : 0,
    config: config.wobbly,
    delay: visible ? delay : 0,
  })
  return <animated.group scale={scale}>{children}</animated.group>
}
```

---

## Actionable for BizzAR

1. **Now (5 lines):** Add `<Sparkles>` on `isTracking` — biggest wow-per-line-of-code ratio
2. **v1 (30 min):** Spring entrance with `useSpring` for contact card panel — `scale: 0 → 1` with `config.wobbly`
3. **v1:** Add `navigator.vibrate(50)` on card detection for haptic feedback (Android Chrome)
4. **v2:** Staggered panel reveal sequence when all modules are added back
5. **v2:** GSAP timeline for sophisticated entrance choreography

---

## Sources
- https://www.react-spring.dev/docs/components/use-spring
- https://drei.docs.pmnd.rs/staging/sparkles
- https://drei.docs.pmnd.rs/animations/use-animations
- https://gsap.com/docs/v3/Plugins/MotionPathPlugin/
- https://github.com/pmndrs/react-three-fiber/discussions (animation patterns)
