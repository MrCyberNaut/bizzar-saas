# AR Target Image Design — Complete Rules
Source: https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-3/
Source: https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-4/

## The Four Rules for a Good Target Image

### 1. Complex (Rich in Detail)
- More dots, lines, shapes, text, logos = more feature points = better tracking
- Plain surfaces with no detail are the worst possible targets
- Even a subtle texture (paper grain, fabric weave) helps

### 2. Non-Repetitive and Non-Symmetrical
- Symmetrical layouts confuse ORB — if the left half looks like the right half, the algorithm can't tell which way the card is oriented
- Repeating patterns (grids, stripes, wallpaper patterns) create identical keypoints that match each other incorrectly
- **Critical for business cards:** Many cards have centered layouts — these are tracking nightmares
- An asymmetric logo in one corner significantly helps

### 3. No Blank Space
- Empty areas contribute zero keypoints
- Blank edges are especially damaging — the tracking algorithm gets confused at frame boundaries
- If your card has a plain white border, crop the target image so the border is removed
- Well-distributed features throughout the image give the tracker fallback points if part of the card is occluded

### 4. High Contrast
- Sharp contrast (black text on white background, dark logo on light area) = strong corners/edges = strong keypoints
- Gradual gradients provide almost no usable keypoints
- **Color does not matter** — the algorithm converts to grayscale. Only luminance values matter.

## What Doesn't Help
- Color (converted to grayscale)
- Smooth gradients (no sharp transitions)
- Resolution above ~1000px (diminishing returns)

## Technical Specs
- **Minimum resolution:** 1000px per dimension
- **Optimal aspect ratio:** Square (1:1). Extreme landscape/portrait reduces tracking surface.
- **If card is horizontal:** Use the landscape image. The tracker handles both orientations.

## The e-cell MIT Manipal Card — Analysis
Problem areas:
- Left half: tan/beige uniform background — ZERO keypoints in this area
- White border: non-unique keypoints
- Centered "e" logo: somewhat symmetrical

Good areas:
- Right half: dark blue with white "e" logo + "e-cell MIT. MANIPAL" text — good contrast
- Name + title + phone/email text on left — some keypoints here

Score estimate: ~4/10 for tracking quality. Functional but marginal.

## Fixes for the e-cell card
1. Crop to remove white border
2. Compile .mind from a photo taken under flat, non-specular lighting (diffuse, overcast)
3. Consider adding the QR code or a decorative element to the tan area
4. Ensure the photo is sharp (no motion blur)

## Card Template Design Recommendations for BizzAR SaaS
When providing card templates to users:
- Always put a unique asymmetric element (logo, photo) in one corner
- Use high-contrast text/background combinations
- Avoid centered/symmetrical layouts
- Add subtle texture or pattern to plain background areas
- Avoid full-bleed gradients as background
- Make the QR code part of the card design — it adds keypoints AND serves a purpose

## Scoring Metrics (from mindar.org tool)
MindAR provides a target image analyzer that scores:
1. **Feature Distribution** — divides image into regions, penalizes empty regions
2. **Feature Uniqueness** — penalizes similar-looking feature points
3. **Image Dimension** — must exceed 1000px

Warning: "Don't trust the scores blindly" — use as a reference only.

## Reflective/Glossy Surfaces
Not addressed in the official docs. From our engineering experience:
- Gloss lamination creates specular highlights that shift position as camera angle changes
- These false highlights confuse ORB (it sees them as corners)
- Our WebGL preprocessing pipeline (glare suppression shader) mitigates this
- Matte lamination is the hardware solution — recommend it to users in the dashboard
