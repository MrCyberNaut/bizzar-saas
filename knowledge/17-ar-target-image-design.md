# AR Target Image Design — Research Guide for BizzAR

## What is Feature-Based Image Tracking?

AR image tracking (NFT — Natural Feature Tracking) works by:
1. Extracting **keypoints** from the target image — corners, edges, unique local patches
2. Computing a **descriptor** for each keypoint — a compact fingerprint of the surrounding pixel patch
3. At runtime: finding the same keypoints in the camera frame and computing a homography transform

The quality of your AR experience is **directly determined by the quality of the features the algorithm can extract from your target image**. This is a constraint on card design that must be understood before designing for BizzAR.

---

## The Core Principle: What the Algorithm Sees

The tracking engine converts your image to **grayscale** before processing. Color is irrelevant to detection — contrast in luminance is everything.

A feature point is detected at a location where there are lots of **rapid local changes** — corners, edges meeting at angles, small unique detail patterns. The algorithm then describes this location by sampling the pixel values around it, creating a fingerprint.

For two images to match, their feature fingerprints must be **unique enough to distinguish** from all other fingerprints in the image. If many locations look similar to each other (repetitive patterns), matching becomes ambiguous and tracking fails.

---

## Rule 1: Feature Richness (More is Better)

**What creates good features:**
- Sharp corners (text, logos, geometric design elements)
- High-frequency texture (fine patterns, grain, illustration detail)
- Diagonal edges (not just horizontal/vertical — those are less distinctive)
- Complex overlapping shapes
- Irregular organic elements (hand-drawn art, complex logos)

**What creates few features:**
- Large flat color areas (blank white/black space)
- Gradients (smooth transitions → no sharp changes → no keypoints)
- Very fine, uniform textures (noise-like patterns look the same everywhere → ambiguous matching)
- Blurry or low-resolution areas (JPEG artifacts smear edges)

**Practical guideline:** A target image should have **visible complexity in at least 70% of its area**. Avoid designs with large monochromatic regions.

---

## Rule 2: Avoid Symmetry

**This is the most common mistake in business card AR.**

Symmetry causes two categories of failure:

### Rotational symmetry (180° flip ambiguity)
If a card design looks the same when rotated 180°, the tracking engine cannot determine the correct orientation. The AR overlay will intermittently flip, appearing correctly oriented and then inverted, depending on which orientation the algorithm converges on.

Example cards that fail:
- Centered logo on plain background
- Symmetric geometric patterns (circles, diamond grids)
- Left-right mirrored text layouts

### Reflective (bilateral) symmetry
If the left half mirrors the right half, features on the left will match features on the right, causing ambiguity in the horizontal position estimate. The AR object will "swim" between two horizontal positions.

**Fix:** Ensure the card has at least one clearly asymmetric element — a logo off-center, text running to one edge, a unique design element in one corner, or an illustration that breaks symmetry.

---

## Rule 3: Feature Distribution (Spread = Stability)

The tracking engine estimates a **homography transform** (a projective mapping) from matched feature pairs. The mathematical stability of this estimation depends on how well the features span the image.

A homography requires 4+ non-collinear point matches. The more spread out the matched points are across the entire image area, the more overdetermined the system is, and the more accurate and stable the pose estimate.

**Bad distribution pattern:** All features clustered in the center (minimalist card with centered text on plain background). The outer corners of the card have no coverage → the engine cannot reliably constrain the perspective transform → the AR overlay wobbles.

**Good distribution:** Features present in all quadrants — top-left, top-right, bottom-left, bottom-right, and center.

**Pictarize's distribution score** divides the image into a grid of regions and deducts points for each empty region. Aim for no large empty zones.

---

## Rule 4: Contrast (Luminance, Not Color)

Since the algorithm works in grayscale, **color contrast does not help** unless it translates to luminance contrast.

| Card design | Grayscale result | Feature quality |
|---|---|---|
| Dark blue text on black | Nearly invisible | Very poor |
| Yellow text on white | Medium gray on white | Poor |
| Black text on white | High contrast | Excellent |
| Red photo on dark green | Both similar luminance | Poor |
| Complex photo | Wide luminance histogram | Excellent |

**Test:** Convert your card design to grayscale before compiling. If the image looks flat and lacks contrast in grayscale, it will track poorly.

**Histogram rule (from Vuforia):** A wide, flat luminance histogram = good target. A narrow, spiky histogram = poor target. The algorithm needs a full range of intensity values distributed across the image.

---

## Rule 5: Uniqueness (No Repeated Patterns)

Repeated/tiled patterns are tracking enemies because:
- The same descriptor appears at many locations
- The matching algorithm generates many false correspondences
- The homography solver cannot determine the correct one from the ambiguous matches

**Examples of bad patterns:**
- Repeating dot grid or halftone background
- Stripe patterns (the same edge pattern repeats vertically)
- Crosshatch or mesh backgrounds
- Brick-like patterns

**Why QR codes work:** Despite looking "noisy," QR codes have **asymmetric encoding** — each cell region is unique. However they are visually ugly for a business card and defeat the aesthetic purpose of BizzAR.

**Community insight:** Designs inspired by QR code logic — unique, high-contrast, distributed patterns — are ideal. Abstract illustration that fills the card, complex photography, or information-dense graphic design work well.

---

## Rule 6: Resolution and DPI

### Digital resolution
- **Minimum:** 1000 × 1000 px for reliable detection (Pictarize full-score threshold)
- **Recommended for business cards:** Export at 300 DPI, which for a standard 85mm × 54mm card = **1004 × 638 px minimum**
- **Optimal:** 300 DPI at the final print size, so 2480 × 1575 px (A4 bleed-safe equivalent at 300 DPI)

### Physical print size
- Tracking distance is approximately **10× the physical width** of the target
- Standard business card (85mm wide) → reliable detection up to ~85cm distance
- Users typically hold their phone 30–60cm from a card — this is well within range
- Minimum physical size for reliable detection: **5cm width** (but closer range required)

### DPI impact on tracking
- 300+ DPI: full feature richness, stable at normal phone distance
- 150 DPI: reduced features, user must hold phone closer
- 72 DPI (screen resolution): very poor — tracking requires near-contact

---

## Specific Guidelines for Business Cards

### Layout zones and tracking impact

```
┌─────────────────────────────────────────────────────┐
│  [DANGER ZONE]     Plain background = zero features  │
│                                                      │
│         Logo (centered)                              │  ← Only features are
│                                                      │     in a small center
│         Name                                         │     region — BAD
│         Title                                        │
│                                                      │
│  [DANGER ZONE]     Plain background = zero features  │
└─────────────────────────────────────────────────────┘

vs.

┌─────────────────────────────────────────────────────┐
│ [Complex illustration / texture fills corners]       │
│  ╔═══╗                                              │
│  ║ L ║  Name          [unique asymmetric element]   │
│  ╚═══╝  Title                                       │
│  Contact info         [pattern fills background]    │
│  [texture/art fills all remaining space]             │
└─────────────────────────────────────────────────────┘
```

### Recommendations for BizzAR users

| Element | Tracking impact | Recommendation |
|---|---|---|
| **Background** | Critical | Use textured, illustrated, or photographic backgrounds instead of flat colors |
| **Logo** | Good if complex | Complex logos with fine detail are excellent anchors |
| **Text** | Good | Dense text (multiple lines) contributes many edge features |
| **White space** | Bad | Avoid large blank margins — push design to the edges |
| **Centered single element** | Bad | Off-center asymmetric layouts track better |
| **Symmetrical design** | Bad | Breaks orientation detection |
| **QR code** | Excellent (but ugly) | Use as a small design element if possible |
| **Photo** | Usually excellent | Complex photos with faces, textures, backgrounds work very well |
| **Gradients** | Bad | Replace with textured gradients or illustrated fills |
| **Dark on dark** | Terrible | Ensure luminance contrast, not just color contrast |
| **Glossy print** | Risky | Matte finish prevents specular highlights corrupting tracking |

---

## Common Real-World Failures

### Failure 1: "Detective card" — only text on white
A white card with name, phone number, and email. Only features are along character edges. No features in 80% of the card area. KPM feature count: ~50–80. Tracking unstable.

**Fix:** Add a colored or illustrated background, move the logo to a corner, add a texture to fill white space.

### Failure 2: "Corporate symmetry" — centered logo mirrored
Dark background, centered embossed-looking logo with bilateral symmetry. The engine finds the logo features but cannot determine up from down or left from right. AR overlay flips.

**Fix:** Add an asymmetric text block, a unique badge, or rotate the logo slightly off-center.

### Failure 3: "Halftone background" — repeating dot pattern
Trendy half-tone or polka-dot background. Massive feature count (~2000) but nearly all features look identical. Matching produces hundreds of false correspondences. Pose estimation is unstable.

**Fix:** Replace with unique illustrated texture or photo.

### Failure 4: "Minimal luxury" — matte black on black with gold foil
Very common in luxury branding. Grayscale conversion reveals essentially a black image with minimal luminance variation. Feature count: ~10–30. Tracking fails completely.

**Fix:** This design style is fundamentally incompatible with feature tracking. Options: (1) Use a QR code instead; (2) Add an illustrated insert area; (3) Add a high-contrast back side for tracking.

---

## What Actually Works (Real-World Successes)

- **Illustrated cards** — hand-drawn artwork filling the card face → excellent (300–800+ features)
- **Photography** — headshot or product photo as background → very good (400–1000+ features)
- **Dense information design** — lots of text, icons, contact details → good (200–400 features)
- **Complex abstract design** — geometric art with many unique intersections → good
- **Map background** — street maps or topographic lines → excellent (many unique intersections)

---

## QR Code vs. Illustrated Target — Comparison

| Property | QR Code | Well-designed Illustrated Card |
|---|---|---|
| Feature count | ~1500–3000 | 300–800 |
| Feature uniqueness | Very high | High |
| Feature distribution | Even | Depends on design |
| Orientation robustness | Moderate (has position markers) | Good if asymmetric |
| Visual aesthetics | Poor | Excellent |
| Physical size sensitivity | Less sensitive | Standard |
| Designer flexibility | None | Full |
| Detection distance | High | Standard |

**Verdict for BizzAR:** Design-first approach with trackability validation (via the Compiler API) is the right UX. Show users their feature heatmap and score when uploading.

---

## The Pictarize Three-Dimension Score

Pictarize (by the MindAR author) built a trackability analyzer at https://pictarize.com/image-analyzer/ that scores images on:

| Dimension | How computed | What it means |
|---|---|---|
| **Distribution** | Divides image into regions; deducts score per empty region | Even spread of features across the card |
| **Uniqueness** | Computes similarities between feature descriptors; penalizes similar descriptors | Features are distinguishable from each other |
| **Resolution** | Full score if > 1000px per dimension and reasonable aspect ratio | Image quality sufficient for stable tracking |

A good target scores well on all three. A card with good uniqueness but poor distribution will track unstably. A card with high distribution but low uniqueness will produce ambiguous matches.

**Build the same three signals into BizzAR's trackability checker** — compute them from the `dataList` returned by `compileImageTargets`.

---

## Actionable for BizzAR

### For the card editor UX
1. **Show the feature heatmap** immediately after card background upload — overlay green dots from the Compiler output on a preview canvas
2. **Show the three-dimension score** (distribution, uniqueness, resolution) — compute from the `dataList`
3. **Block AR activation** if total KPM feature count < 100 with a clear message: "Your card design has too few trackable features. Add more visual complexity."
4. **Warn** between 100–300 features: "Tracking may be unstable in low light. Consider adding more visual detail."
5. **Display tips inline:** When score is low, show the specific problem (e.g., "Large blank areas detected in corners — consider adding a textured background")

### For the design guide (user-facing content)
Provide a visual guide to users:
- "Good" examples: illustrated cards, photo backgrounds, dense design
- "Bad" examples: minimal white space, centered symmetrical logos
- "Fix" examples: showing the same card improved

### Card template design
Build BizzAR's built-in card templates to be tracking-optimized by default:
- All templates must have textured/illustrated backgrounds
- No template should have more than 30% plain/gradient background
- Default layouts are asymmetric
- All templates should score ≥ "good" (300+ KPM features) when compiled

### Physical print advice
Include in BizzAR's print/export guidance:
- Print at 300 DPI minimum
- Use matte (not glossy) lamination — glossy causes specular tracking failures under bright lights
- Minimum card width: 85mm (standard business card is right at the lower boundary — ensure design fills it)
- Users scanning: hold phone 20–60cm from card, steady hand

---

## Sources

- [How to Choose Part 1 — Feature points theory](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-1/)
- [How to Choose Part 2 — Feature quality](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-2/)
- [How to Choose Part 3 — Distribution and uniqueness](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-3/)
- [How to Choose Part 4 — Pictarize analyzer](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-4/)
- [Pictarize image analyzer tool](https://pictarize.com/image-analyzer/)
- [How to Choose Part 3 — Medium](https://medium.com/web-augmented-reality-development/how-to-choose-a-good-target-image-for-tracking-in-ar-part-3-3d70a958a33a)
- [Vuforia Best Practices](https://developer.vuforia.com/library/objects/best-practices-designing-and-developing-image-based-targets)
- [Vuforia Image Optimization Techniques](https://developer.vuforia.com/library/objects/image-targets-optimization-techniques)
- [AR.js Image Tracking Docs](https://ar-js-org.github.io/AR.js-Docs/image-tracking/)
- [MindAR Compile Target Images](https://hiukim.github.io/mind-ar-js-doc/quick-start/compile/)
- [Introduction to AR.js NFT](https://web.navan.dev/posts/2020-08-01-Natural-Feature-Tracking-ARJS)
- [NFT Marker Creator Tool](https://ar-js-org.github.io/NFT-Marker-Creator/)
