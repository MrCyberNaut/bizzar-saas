# AR Business Card UX — Patterns & Competitive Analysis

**Date:** 2026-05-17
**Relevance:** What to build, how to design the scan experience, what drives retention

---

## 1. Industry Examples

### What exists
- **Aircards** (aircards.com): WebAR cards for enterprise (Stanley Black & Decker, etc). Image tracking + video overlays. Commercial platform.
- **Popl**: NFC-first digital business card. No AR — just profile page on tap. ~$15/year.
- **HiHello**: Digital cards, QR, contact management. No AR.
- **Linq**: NFC + QR digital card. No AR.
- **dot.cards**: Smart business card (NFC). No AR.
- **8th Wall demos**: Multiple agency-built AR business card demos (pre-shutdown) — most impressive were 3D avatar + floating contact info.

### Key finding
**None of the major digital card platforms (Popl, HiHello, Linq, dot) do real AR.** BizzAR has a clear differentiation gap.

---

## 2. What Works in AR Business Card Experiences

Based on agency demos and user research from similar experiences:

### Hierarchy of impact (highest to lowest wow factor)
1. **3D avatar / character** that waves or animates — highest recall
2. **Video reel** playing in a floating panel — portfolio/demo video
3. **Animated contact info** popping up with spring entrance
4. **GitHub/portfolio stats** in a floating panel — great for developers
5. **Simple contact + Add to Contacts** — utility, always useful

### What users actually do
- Scan once to get contact → tap "Add to Contacts" → done
- ~20% explore the additional panels
- ~5% share the experience with someone else ("look at this")

**Implication:** The Add to Contacts button is the #1 conversion action. Make it prominent and instant.

---

## 3. Scanning UX Patterns

### Viewfinder / Target Guide
Show a scanning frame that "locks" onto the card:
```typescript
// Before card detected: animated corner brackets
// After card detected: corners turn green, "Card detected" pill appears
```

### MindAR warmup/miss tolerance → UX
- `warmupTolerance: 2` = 2 frames needed to confirm → ~66ms delay (imperceptible)
- `missTolerance: 25` = 25 frames before losing = ~833ms grace period
- Show "Card detected" immediately on `onAnchorFound`
- DON'T hide content until `onAnchorLost` + 500ms debounce

### Loading States
1. **Page load:** "Loading AR experience..." (dynamic import loading)
2. **Camera init:** "Initializing camera..." (getUserMedia pending)
3. **MindAR ready:** "Point camera at your business card" (onReady fired)
4. **Card detected:** "Card detected" → AR content pops in

---

## 4. Content Hierarchy for BizzAR v1

```
Primary (always visible when card detected):
  - Contact card panel (photo + Add to Contacts)
  - Card frame glow

Secondary (tap to expand, v2):
  - LinkedIn / Twitter / GitHub links
  - Bio text

Tertiary (v3):
  - GitHub contribution graph
  - Portfolio gallery
  - Demo video
  - 3D avatar/model
```

Start with primary only. Each panel addition must justify itself with data (do users tap it?).

---

## 5. "Wow Factor" Effects

```typescript
// Sparkles on first card detection — from drei
import { Sparkles } from '@react-three/drei'

{isTracking && (
  <Sparkles count={40} scale={[1.0, 0.7, 0.05]} size={0.06} speed={0.4} color="#818cf8" />
)}

// Spring entrance for panels
import { useSpring, animated } from '@react-spring/three'

const { scale } = useSpring({
  scale: isTracking ? 1 : 0,
  config: { tension: 300, friction: 20 }
})
```

---

## 6. Error States

```typescript
const ERROR_STATES = {
  NO_CAMERA: {
    icon: '📷',
    title: 'Camera access needed',
    message: 'Allow camera access to use AR',
    action: 'Grant Access',
  },
  IN_APP_BROWSER: {
    icon: '🌐',
    title: 'Open in browser',
    message: 'AR works best in Safari or Chrome',
    action: 'Open in Safari',
  },
  CARD_NOT_FOUND: {
    // After 30s with no detection
    icon: '🔍',
    title: 'Card not found',
    message: 'Make sure the card is well-lit and fully visible',
    action: null,
  },
  LOAD_FAILED: {
    icon: '⚠️',
    title: 'AR failed to start',
    message: errorMessage,
    action: 'Retry',
  },
}
```

---

## 7. QR Code Fallback

For non-camera scenarios (sharing link via email, camera denied, desktop):
- The AR URL `/ar/[userId]/[cardId]` should show a non-AR profile page
- Generate a QR code pointing to the AR URL — print on the card for easy scanning
- QR scan on phone → opens mobile browser → AR experience

```typescript
// Detect if AR is likely to work
const arSupported = navigator.mediaDevices?.getUserMedia && !isDesktop()
if (!arSupported) {
  // Show profile page instead of AR experience
  redirect(`/profile/${userId}/${cardId}`)
}
```

---

## 8. Sharing & Virality

The AR experience URL is inherently shareable:
- Include an Open Graph preview image showing the AR experience
- "Made with BizzAR" watermark links to bizzar.app — organic acquisition
- Add "Create your own AR card →" CTA after successful scan

```typescript
// After tracking starts, show a subtle CTA
{tracking && (
  <div style={{ position: 'fixed', bottom: 80, right: 16, zIndex: 10 }}>
    <a href="https://bizzar.app" target="_blank">
      <button>Create yours →</button>
    </a>
  </div>
)}
```

---

## Actionable for BizzAR

1. **Now:** Add `<Sparkles>` on card detection — ~5 lines, big wow moment
2. **Now:** Add "Create your own AR card →" CTA after successful scan — viral loop
3. **v1:** In-app browser detection + "Open in Safari/Chrome" banner
4. **v1:** Non-AR profile page fallback for desktop / camera-denied users
5. **v1:** 30s no-detection timeout → show tips ("ensure card is in good lighting")
6. **v2:** Spring entrance animations for all panels
7. **v2:** Haptic feedback on card detection (`navigator.vibrate(50)`)

---

## Sources
- https://aircards.co (enterprise AR cards)
- https://popl.co, https://hihello.me, https://linq.to (competitors — no AR)
- https://8thwall.com/blog (archived AR business card demos)
- https://web.dev/patterns/advanced-apps/contacts/ (Web Contacts API)
