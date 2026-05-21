# 02 — The One Euro Filter (1€ Filter)

**Date:** 2026-05-19
**Topic:** The pose-smoothing algorithm MindAR uses internally. How it works, tuning, why MindAR's defaults map the way they do.

---

## TL;DR

The One Euro Filter is a **speed-adaptive low-pass filter** designed for noisy real-time interactive signals (mouse, pose, joint tracking, AR markers). It has two knobs:
- `min_cutoff` (a.k.a. MindAR's `filterMinCF`) — minimum cutoff frequency, sets the **stationary** smoothing level.
- `beta` (a.k.a. MindAR's `filterBeta`) — speed coefficient, sets how fast the filter "lets go" when the signal moves quickly.

Originally published at CHI 2012 by Géry Casiez, Nicolas Roussel, Daniel Vogel. Used in MediaPipe, ARKit-adjacent pipelines, mouse trajectory smoothers, joint tracking, and MindAR.

---

## Why a Speed-Adaptive Filter?

Humans are **very sensitive to jitter when input is slow** (e.g. a still cursor wobbling pixel-by-pixel is awful) but **very sensitive to lag when input is fast** (a cursor that drifts behind your hand is awful). Conventional fixed-cutoff low-pass filters have to pick one or the other.

The 1€ filter's insight: **make the cutoff frequency a function of velocity.**
- Slow signal → low cutoff → smooth, jitter-free, but lagged (user can't tell because they're not moving).
- Fast signal → high cutoff → low lag, some jitter (user can't tell because they're moving fast).

> "people are very sensitive to jitter and not latency when moving slowly, but as movement speed increases, people become very sensitive to latency and not jitter."
> — Source summary from [Casiez et al., CHI 2012](https://inria.hal.science/hal-00670496/)

---

## The Algorithm

### Step 1: Smoothing factor from cutoff frequency

```
α(t_e, cutoff) = r / (r + 1)
where r = 2π · cutoff · t_e
```

`t_e` is the elapsed time between samples (seconds). This converts a cutoff frequency in Hz to an exponential-smoothing alpha.

### Step 2: Compute the derivative (signal velocity)

```
dx_i = (x_i − x̂_{i−1}) / t_e        // raw derivative
dx̂_i = α(t_e, d_cutoff) · dx_i      // smoothed with fixed d_cutoff (default 1.0 Hz)
       + (1 − α) · dx̂_{i−1}
```

### Step 3: Compute the *adaptive* cutoff frequency

```
cutoff = min_cutoff + β · |dx̂_i|
```

The faster the signal is moving, the higher the cutoff — the filter becomes more responsive.

### Step 4: Filter the actual signal

```
α_signal = α(t_e, cutoff)
x̂_i = α_signal · x_i + (1 − α_signal) · x̂_{i−1}
```

### Python pseudocode (from Jaan Tollander's reference impl)

```python
import math

def smoothing_factor(t_e, cutoff):
    r = 2 * math.pi * cutoff * t_e
    return r / (r + 1)

def exponential_smoothing(a, x, x_prev):
    return a * x + (1 - a) * x_prev

class OneEuroFilter:
    def __init__(self, t0, x0, dx0=0.0, min_cutoff=1.0, beta=0.0, d_cutoff=1.0):
        self.min_cutoff = min_cutoff
        self.beta = beta
        self.d_cutoff = d_cutoff
        self.x_prev = x0
        self.dx_prev = dx0
        self.t_prev = t0

    def __call__(self, t, x):
        t_e = t - self.t_prev
        # filter the derivative
        a_d = smoothing_factor(t_e, self.d_cutoff)
        dx = (x - self.x_prev) / t_e
        dx_hat = exponential_smoothing(a_d, dx, self.dx_prev)
        # adaptive cutoff
        cutoff = self.min_cutoff + self.beta * abs(dx_hat)
        a = smoothing_factor(t_e, cutoff)
        x_hat = exponential_smoothing(a, x, self.x_prev)
        # persist
        self.x_prev, self.dx_prev, self.t_prev = x_hat, dx_hat, t
        return x_hat
```

Source: [jaantollander/OneEuroFilter on GitHub](https://github.com/jaantollander/OneEuroFilter).

---

## Parameter Tuning Procedure (per the paper)

1. **Set β = 0.** This disables speed adaptation — you now have a vanilla low-pass.
2. **Find a `min_cutoff` low enough that the still-signal jitter is acceptable.** Start at 1.0 Hz, drop in halves until jitter is OK. Lag will be bad — that's fine.
3. **Now raise β** until the lag during motion is acceptable. β has no fixed scale; what matters is `β · velocity` is comparable to `min_cutoff`.

The paper notes: "If high speed lag is a problem, increase β; if slow speed jitter is a problem, decrease min_cutoff."

---

## Mapping to MindAR's parameter names

| MindAR name | 1€ filter name | MindAR default | Filter-paper-typical default |
|---|---|---|---|
| `filterMinCF` | `min_cutoff` (fc_min) | `0.001` Hz | `1.0` Hz |
| `filterBeta` | `β` | `1000` | `0.0` (start) |
| (not exposed) | `d_cutoff` | hardcoded | `1.0` Hz |

**Note** the MindAR default `filterMinCF: 0.001` is *three orders of magnitude lower* than the typical filter starting point — MindAR has dialed in a very heavily smoothed baseline because its pose estimates are noisy. The `filterBeta: 1000` is correspondingly very high to compensate during motion. This explains why small changes to either parameter can produce wildly different feels.

---

## Why MindAR's Defaults Feel "Floaty"

With `filterMinCF = 0.001` Hz, the cutoff at zero velocity is essentially DC — the filter barely lets the signal through. The only thing keeping content responsive is `filterBeta = 1000` kicking the cutoff up the moment the user moves. This creates two failure modes:
- **Sub-threshold motion** (tiny hand tremor): not enough velocity to trigger β, so the filter is in deep-smoothing mode → content feels *glued* and slightly behind.
- **Just-stopping motion**: velocity drops to 0 instantly, but the filter's velocity estimate has its own lag (`d_cutoff = 1.0`), so the content keeps drifting briefly after the user stops → "settling" wobble.

---

## Actionable for BizzAR

1. **Implement a custom OneEuroFilter wrapper** on top of MindAR's output if MindAR's exposed knobs don't suffice. The math is ~30 lines of JS. Apply it per-axis (x, y, z position + quaternion components, or better, log-quaternion / axis-angle for rotation).
2. **Tune by content type:**
   - Static panels (text, contact card): bias toward smoothing → `filterMinCF: 0.0001`, `filterBeta: 5000`. Tolerate lag.
   - Animated 3D models / GitHub graph: bias toward responsiveness → `filterMinCF: 0.01`, `filterBeta: 500`. Tolerate jitter (the animation hides it).
3. **Consider a second-stage filter** with a fixed cutoff applied to the smoothed pose specifically for "wobble-sensitive" UI like long text — different content can use different filtering even if the parent pose is the same.
4. **Expose `d_cutoff` if you fork MindAR.** Lowering `d_cutoff` to ~0.5 Hz reduces post-stop settling drift at the cost of slightly more sluggish response to velocity changes.
5. **Predictive offset (advanced):** Combine 1€ output with a small forward-prediction term (`x̂ + k · dx̂`) to compensate for inherent display latency. ARKit does this; MindAR doesn't.

---

## Sources

- [Casiez, Roussel, Vogel — 1€ Filter, CHI 2012 (PDF)](https://inria.hal.science/hal-00670496/PDF/CHI2012-casiez.pdf)
- [Noise Filtering Using 1€ Filter — Jaan Tollander](https://jaantollander.com/post/noise-filtering-using-one-euro-filter/)
- [jaantollander/OneEuroFilter (Python reference)](https://github.com/jaantollander/OneEuroFilter)
- [casiez/OneEuroFilter (author's repo)](https://github.com/casiez/OneEuroFilter)
- [Needle Engine OneEuroFilter (JS impl)](https://engine.needle.tools/docs/api/@needle-tools/engine/3.34.1-alpha/classes/engine_api.OneEuroFilter.html)
- [1eurofilter on npm](https://www.npmjs.com/package/1eurofilter)
- [OneEuro Filter overview — Mohamed Ali Rashad](https://mohamedalirashad.github.io/FreeFaceMoCap/2021-12-25-filters-for-stability/)
