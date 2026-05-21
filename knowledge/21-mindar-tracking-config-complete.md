# MindAR Tracking Configuration — Complete Reference
Source: https://hiukim.github.io/mind-ar-js-doc/quick-start/tracking-config/

## All Parameters

### filterMinCF (Cutoff Frequency)
- **Default:** 0.001
- **Effect of decreasing:** Reduces jitter. Content appears smoother/more stable.
- **Cost of decreasing:** Adds delay. Content lags behind real camera movement.
- **Effect of increasing:** Reduces lag. Content snaps to card instantly.
- **Cost of increasing:** More jitter. Content shakes even when card is still.
- **Our current value:** 0.001 (BALANCED), 0.0001 (SMOOTH), 0.01 (FAST)

### filterBeta (Speed Coefficient)
- **Default:** 1000 (NOTE: MindAR docs say 1000, not 100 as we originally used)
- **Effect of increasing:** Reduces delay. Content follows fast movements better.
- **Cost of increasing:** More jitter at low speeds.
- **Effect of decreasing:** Smoother at low speeds.
- **Cost of decreasing:** More lag during fast movement.
- **Interaction:** filterMinCF and filterBeta fight each other. You cannot reduce both jitter AND lag simultaneously.
- **Our current value:** 20 (BALANCED), 5 (SMOOTH), 100 (FAST)

### warmupTolerance
- **Default:** 5 frames
- **Meaning:** Number of consecutive detection frames required before `onAnchorFound` fires.
- **Effect of decreasing to 1:** Faster initial lock. May get false positives (phantom detections).
- **Effect of increasing:** Slower initial lock but more confident.
- **Recommendation:** 2–3 for production (fast enough, stable enough).
- **Our current value:** 5 (SMOOTH), 3 (BALANCED), 2 (FAST)

### missTolerance
- **Default:** 5 frames
- **Meaning:** Number of consecutive missed detection frames before `onAnchorLost` fires.
- **Effect of decreasing:** AR content disappears faster when card leaves frame. More responsive.
- **Effect of increasing:** AR content lingers longer. Hides brief tracking failures (ghosting).
- **Recommendation:** 5–10 for UX. Lower if you want snappy on/off. Higher if card moves slightly out of frame briefly.
- **Our current value:** 10 (SMOOTH), 5 (BALANCED), 3 (FAST)

## IMPORTANT: Default filterBeta discrepancy
MindAR docs say default filterBeta=1000. react-three-mind may use a different default.
Our BALANCED profile uses beta=20 which is much lower than MindAR default.
EXPERIMENT: Try beta=500-1000 for the FAST profile to match MindAR default behavior.

## One Euro Filter Background
Both filterMinCF and filterBeta are parameters of the **One Euro Filter**, a signal smoothing algorithm.
- Paper: "One Euro Filter: A Simple Speed-based Low-pass Filter for Noisy Input in Interactive Systems" (Casiez et al., 2012)
- The filter adapts its cutoff frequency based on signal speed: slow movement = heavy smoothing, fast movement = less smoothing
- This is why filterBeta matters: higher beta = more aggressive cutoff adaptation at speed

## A-Frame Usage
```html
<a-scene mindar-image="filterMinCF:0.1; filterBeta: 10; warmupTolerance: 2; missTolerance: 5">
```

## react-three-mind Usage
```tsx
<ARView
  filterMinCF={0.001}
  filterBeta={20}
  warmupTolerance={3}
  missTolerance={5}
>
```
