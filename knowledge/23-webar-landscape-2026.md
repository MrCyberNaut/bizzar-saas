# WebAR Landscape 2026 — After 8th Wall
Source: https://www.kivicube.com/post/augmented-reality-after-8th-wall-shutdown-your-2026-guide-to-the-right-webar-platform/
Source: Search synthesis

## 8th Wall Shutdown — What Happened
- **Platform access ended: February 28, 2026**
- Hosted campaigns viewable until February 28, 2027
- After 2027: all content + links permanently disabled, all data deleted
- Impact: largest WebAR developer platform shutdown. Thousands of live AR experiences broken.

## The Gap This Creates
8th Wall had:
- Best-in-class image tracking (proprietary algorithm, far better than MindAR ORB)
- Native iOS + Android support
- Developer SDK + no-code tools
- Enterprise pricing ($99-$250+/month)

Nobody in the open-source space matches 8th Wall's tracking quality today.

## Current Alternatives (Paid Platforms)

### Blippar WebAR SDK
- Most common migration path for developer teams
- Blippbuilder for no-code teams
- Good image tracking
- Enterprise pricing

### Zapworks
- Voted "best overall alternative" in multiple comparisons
- Designer + developer tools
- Marker-based + markerless AR

### Kivicube
- Face, image, world, body tracking
- Pure WebAR (WebGL + WebXR, iOS Safari + Android Chrome)
- Drag-drop editor + APIs
- Image tracking via custom OpenCV engine
- Free tier available

### AR Code
- AR QR codes — trigger AR from QR scan
- No full image tracking (QR marker only)
- Strong for business cards, packaging
- Per-QR pricing

### ARLOOPA Studio
- Marker-based image tracking
- Good for posters, packaging, business cards
- Web link publishing (no app)

### Artivive
- Artist-focused, not SaaS/dev
- Good for printed materials
- Limited module/content flexibility

### Hololink
- Custom OpenCV engine
- No-code focused

## Open Source Options

### MindAR (hiukim/mind-ar-js)
- Best open-source image tracking for browsers
- ORB-based (inferior to commercial solutions but free)
- A-Frame + Three.js support
- Maintained but slow development
- react-three-mind wrapper (last meaningful update 2023)

### AR.js
- Older, more established open source
- Marker-based (ArUco/Hiro) and NFT (Natural Feature Tracking)
- ARTOOLKIT under the hood
- Less polished than MindAR

### WebAR.rocks
- Neural network-based (NOT ORB)
- Better on textureless objects
- Needs custom training per object class
- Not plug-and-play for arbitrary images

## BizzAR's Competitive Position
- 8th Wall dead → no dominant open-source platform exists
- Paid platforms are $99-$250+/month, enterprise-focused
- **The gap:** A consumer/prosumer WebAR SaaS with a drag-drop module editor doesn't exist
- Closest: AR Code (QR trigger only), Artivive (artist-focused, no modules)
- **Our advantage:** Physical card trigger + modular panel editor + Supabase backend = unique combination

## What Would Make BizzAR's Engine Competitive
1. **Better tracking than MindAR ORB** — CLAHE preprocessing, or ML-based detector
2. **iOS compatibility** — captureStream workaround for Safari
3. **Target image quality gate** — warn users before publishing with bad cards
4. **The module editor** — nobody has this for AR business cards

## Market Context
- Digital business card market: $217M in 2026 → $332M by 2031 (8.86% CAGR)
- Asia Pacific fastest growing at 10.43% CAGR — our core market (India/SEA)
- Progressive web apps segment growing at 9.11% — aligns with our stack
- SME segment 38.5% of market — our primary customer
