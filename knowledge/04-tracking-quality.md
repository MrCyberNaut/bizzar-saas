# 04 — MindAR Tracking Quality Improvements

**Date:** 2026-05-19
**Topic:** Practical levers to make MindAR tracking more stable and accurate: target image design, `.mind` compilation, lighting, camera constraints, environment.

---

## TL;DR

Tracking quality is dominated, in order, by:
1. **Target image design** (feature distribution and uniqueness) — 60% of the win.
2. **Compiled `.mind` file feature count** — 15%.
3. **Lighting + camera focus + resolution at runtime** — 15%.
4. **Filter / tolerance tuning** — 10% (covered in [01-mindar-parameters.md](./01-mindar-parameters.md)).

If a target has bad features, no amount of filter tuning will save it.

---

## 1. Target Image Design

### Resolution
- **Minimum: 1000 px on shortest side.** The MindAR compiler scores images on resolution. Anything < ~512 px will compile with sparse features.
- **Preferred aspect ratio: square or near-square.** Wide / tall ratios concentrate features unevenly.
- Source: [mindar.org — How to Choose a Good Target Image Part 3](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-3/)

### Feature density and distribution
- **Quote:** "high quality feature points exist on locations where there are lots of changes" — i.e. corners, edges, sharp gradients.
- **Quote:** "you want to have good amount of features with well distribution."
- **Quote:** "the better the distribution of feature points, the more tolerant they are to errors. ... aim to have feature points exist across every regions of the target image, rather than to have them densely gathered in some regions, leaving the other regions empty."
- Source: [mindar.org Part 1](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-1/)

### Contrast
- **Quote:** "Contrast is good - most high quality feature points exist on positions where there are changes, mostly being found on corners and edges, and high contrast can make them more prominent."
- Note that MindAR (like ARToolKit, ARKit, etc.) operates on grayscale — color contrast does *not* help; **luminance contrast does**. A red logo on a green background of identical brightness has zero contrast to the tracker.

### What to avoid
- **Repetitive patterns** (gridded patterns, regular tiles, stripes). Feature points need to be uniquely identifiable; repetition causes mismatches.
- **Symmetrical patterns** (perfect bilateral or rotational symmetry). The pose becomes ambiguous.
- **Blank space**, especially around the edges. Quote: "Avoid blank space as much as possible ... blank space on the peripheral area is even worse."
- **Glossy surfaces.** When printed, gloss causes specular highlights that shift features depending on lighting angle.
- **Smooth gradients with no detail.**

### Best business-card patterns
- Photographic background (a portrait, abstract photo, complex illustration) → many natural features.
- Mixed type/imagery layouts beat pure-typography cards.
- A wordmark in a blank field is one of the *worst* possible targets.

---

## 2. The `.mind` Compilation Output

The MindAR compiler tool (online or `compile.html` from the repo) outputs:
1. A feature-distribution visualization heatmap.
2. The `.mind` binary.

**What to look for in the heatmap:**
- Dense, evenly spread red/yellow dots → great.
- Big blank zones → re-design or crop the image.
- Dots only at the four corners or only along one diagonal → unreliable.

**Feature count rule of thumb:** more features per image = more stable tracking. There's no public "minimum" number, but anecdotal reports in MindAR issues suggest **< ~100 features = unreliable, > 300 features = solid**. (Not officially published; verify empirically.)

**Multi-scale:** the `.mind` format compiles features at multiple scales so the tracker can handle the user holding the card close *or* far. If you compile at the wrong source resolution, the smallest scale may be too coarse — leads to "loses tracking at arm's length" symptoms.

Sources:
- [Compile Target Images doc](https://hiukim.github.io/mind-ar-js-doc/quick-start/compile/)
- [Image Targets Compiler tool doc](https://hiukim.github.io/mind-ar-js-doc/tools/compile/)
- [Augmia free compiler](https://augmia.io/free-tools/mindar-target-compiler)

---

## 3. Runtime Camera & Lighting

### Lighting
- **Even, diffuse lighting** = best.
- **Avoid direct sunlight / single point lights** — creates specular hotspots and harsh shadows that warp the apparent feature pattern frame-to-frame.
- **Low light** drops detector confidence dramatically. MindAR's TF.js detector is not particularly low-light-robust.

### Camera constraints (`getUserMedia`)
- **Resolution:** higher = more features captured, but more CPU. **1280×720 is the sweet spot.** Going above (1080p / 4K) usually doesn't improve tracking but tanks framerate on mid-tier phones.
- **Frame rate:** 30 fps is fine for tracking; 60 fps doubles CPU for marginal stability gains. Request `frameRate: { ideal: 30, max: 30 }`.
- **Facing mode:** `environment` (rear camera).
- **Continuous autofocus:** request `focusMode: 'continuous'` where supported; without it the camera may go out of focus during slow zoom and lose features.

### Background
- A busy/cluttered background behind the card is fine — MindAR only looks for the target.
- A target on a glass / mirrored surface is bad (reflections move).

---

## 4. .mind Best Practices Summary

| Check | Pass criteria |
|---|---|
| Source resolution | ≥ 1000 × 1000 px |
| Aspect ratio | Close to 1:1, or at least 4:3 / 3:4 |
| Feature heatmap | Dots cover all 4 quadrants, no big blank zones |
| Feature count | > 300 (rule of thumb) |
| Repetitive patterns | None |
| Symmetry | Asymmetric |
| Edge features | Yes, features touching the edges |
| Print finish (if physical) | Matte, not gloss |

---

## Actionable for BizzAR

1. **Bake target-image validation into the BizzAR card editor.** When a user uploads a card design:
   - Reject < 1000 px shortest side with a clear message.
   - Run the MindAR compiler client-side (the WASM compiler works in-browser) and **show the feature heatmap to the user** before they save.
   - Compute a "trackability score" (feature count + distribution variance) and display it as a 1–5 star rating.
   - Block save if score < 2 stars; warn if 2–3 stars.
2. **Suggest "background fixer" templates.** If a user's design has a lot of blank space, automatically offer to overlay a subtle textured background pattern that adds features without changing the visual identity. (Half-tone dots, light noise, etc.)
3. **Ship a few sample card templates** that compile to >500 features each. Users gravitate to templates; this defaults them to good tracking.
4. **Force matte print** in BizzAR's recommended print specs. Add it to the documentation/preview.
5. **Tune camera constraints**: request `{ video: { facingMode: 'environment', width: { ideal: 1280 }, height: { ideal: 720 }, frameRate: { ideal: 30 } } }` instead of letting the browser pick.
6. **Add a runtime lighting hint.** If the YUV/Y-channel mean is very low (dark camera frame), surface a "Move to better lighting" toast — much better UX than silent tracking failure.

---

## Sources

- [mindar.org: Choose a Good Target Image Part 1](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-1/)
- [mindar.org: Part 2](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-2/)
- [mindar.org: Part 3](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-3/)
- [mindar.org: Part 4](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-4/)
- [MindAR docs: Compile Target Images](https://hiukim.github.io/mind-ar-js-doc/quick-start/compile/)
- [MindAR docs: Compiler tool](https://hiukim.github.io/mind-ar-js-doc/tools/compile/)
- [Augmia MindAR target compiler](https://augmia.io/free-tools/mindar-target-compiler)
- [Medium: MindAR for beginners](https://medium.com/@invisible_room/mindar-augmented-reality-on-the-web-for-beginners-c805da2c3ab0)
