# react-three-mind — Complete API Reference
Source: https://github.com/tommasoturchi/react-three-mind

## Package Status
- **Version:** 0.3.0 (published May 11, 2025)
- **Stars:** 53 | **Forks:** 12
- **Open issues:** 2 | **Open PRs:** 0
- **Language:** JavaScript 100%
- **Status:** Effectively unmaintained. Last meaningful code change ~2023, v0.3.0 was a dependency bump.
- **TODO in readme:** "Fix Performance Issues", "Fix CI Build" — neither done

## What It Does
Bridges MindAR image tracking with @react-three/fiber. Provides React components that:
1. Manage the camera video background
2. Run MindAR's WASM tracking in a web worker
3. Expose anchor state (found/lost + pose matrix) as R3F groups
4. Apply the camera pose to the R3F scene's projection

## ARView Component — All Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `imageTargets` | string | required | URL to compiled .mind file |
| `maxTrack` | number | 1 | Max simultaneous targets tracked |
| `filterMinCF` | number | 0.1 | One Euro Filter cutoff freq |
| `filterBeta` | number | 1000 | One Euro Filter speed coeff |
| `warmupTolerance` | number | 5 | Frames to confirm detection |
| `missTolerance` | number | 5 | Frames to confirm loss |
| `flipUserCamera` | boolean | true | Mirror video for front camera |
| `autoplay` | boolean | true | Auto-start tracking on load |
| `ref` | RefObject | — | Access to start/stop/pause controls |
| `onReady` | () => void | — | Fires when MindAR initialized |
| `onError` | (err) => void | — | Fires on init error |
| + all R3F Canvas props | | | |

**NOTE:** Default `filterMinCF` is 0.1 in react-three-mind — different from MindAR's own default of 0.001. The react-three-mind default is aggressively smoothing.

**NOTE:** Default `flipUserCamera` is `true` — applies `scale={[-1,1,1]}` to ALL anchor content. This is the source of the invisible content bug.

## ARAnchor Component — All Props

| Prop | Type | Default | Description |
|---|---|---|---|
| `target` | number | required | Index into the .mind file's target list (0-indexed) |
| `onAnchorFound` | () => void | — | Fires when this target is detected |
| `onAnchorLost` | () => void | — | Fires when this target is lost |
| + all R3F Group props | | | |

## Critical Internal Behavior (Line ~32179)

```javascript
// Inside react-three-mind/dist/index.js
DC.jsx("group", { scale: [flipUserCamera ? -1 : 1, 1, 1], children: ... })
```

This wraps ALL ARAnchor children in a group with X scale = -1 when flipUserCamera=true (front camera) or +1 when flipUserCamera=false (back camera).

**Effect:** When scale X = -1, all mesh normals invert → backface culling removes all surfaces → content is invisible.

**Fix:** `AnchorContent` wrapper that applies matching scale to cancel it:
```tsx
<group scale={[flip ? -1 : 1, 1, 1]}>
  {children}
</group>
```
Where `flip = (flipUserCamera === true)` = `(camera === 'front')` in our code.

## ref API (ARView)
```tsx
const arRef = useRef()
<ARView ref={arRef} ...>
// arRef.current.pause()   — stop tracking
// arRef.current.resume()  — resume tracking
// arRef.current.switchCamera() — not well documented
```

## Known Bugs
1. **flipUserCamera X-flip** — content invisible (fixed in our code)
2. **Inline callbacks cause re-render loop** — if any prop changes every render, ARView re-inits. Fixed by useCallback on all handlers.
3. **Performance issues** — mentioned as a TODO, never fixed. Our preprocessor pipeline helps.

## Migration Risk
react-three-mind is a thin wrapper over MindAR. If we need to replace it, the work is:
1. Manually set up MindAR with Three.js API (not A-Frame, not react-three-mind)
2. Apply the camera matrix to R3F's camera manually
3. Handle the video background texture ourselves

This is ~200–300 lines of code. Doable if react-three-mind becomes a blocker.
