# BizzAR — Master TODO

## Phase A — Engine improvements (1-2 weeks) ← CURRENT
Improve tracking on any card without rebuilding MindAR.

- [x] **A1. CLAHE-lite WebGL shader** — extend `camera-preprocess.ts`
      Local histogram equalization per tile before MindAR sees the frame.
      Expected: ~2× more DoG extrema on flat-background cards.
      Reference: fivekogfx (github.com/fiveko/fivekogfx)

- [x] **A2. Phase correlation fallback** — when MindAR NCC drops
      Use WebFFT for fast frame-to-frame translation tracking.
      Prevents full SIFT re-detection on brief occlusions/fast motion.
      Reference: webfft.com + Nature 2025 paper (knowledge/29)

- [ ] **A3. Test on all 3 cards** — measure found/lost ratio before vs after
      E-Cell tan (worst case), E-Cell blue, Inspark.
      Use the existing debug panel counters.

---

## Phase B — Detect-once, track-fast engine (3-4 weeks) ← AFTER first SaaS feature ships
Restructure the detection loop. MindAR demoted to initial detection only.

- [ ] **B1. TrackingEngine class** — wraps MindAR, owns the state machine
      States: DETECTING → LOCKED → TRACKING → RECOVERING → DETECTING
      MindAR only runs in DETECTING state.

- [ ] **B2. NCC tracking layer** — extract MindAR's internal NCC tracker
      Run it independently with configurable confidence threshold.

- [ ] **B3. Phase correlation tracking layer** — recovery before full re-detect
      WebFFT translation estimate → keep AR locked through brief losses.

- [ ] **B4. Optional: js-aruco2 fast-path** — for cards with embedded marker
      Initial lock in 1-2 frames vs 5-10 for SIFT. Opt-in per card.

- [ ] **B5. Publish as open-source** — ArUco bootstrap → NFT handoff pattern
      Not publicly implemented anywhere. Publishable contribution.

---

## SaaS Product (build in parallel, don't let engine block it)
Ship something users can actually touch.

- [ ] **S1. Card upload + .mind compilation flow** — user uploads card image → server compiles .mind → stored in Supabase
- [ ] **S2. Module editor** — drag vCard / Bio / Socials onto panel slots
      Lift module components from bizzar-proto into bizzar/ editor
- [ ] **S3. AR viewer route** — `/ar/[userId]/[cardId]` live with real data
- [ ] **S4. Auth re-enable** — remove bypass before any external sharing
- [ ] **S5. Freemium gate** — Free: 1 card, 2 panels. Pro: unlimited.

---

## Infrastructure
- [ ] Supabase keep-alive (GitHub Actions cron, ping every 3 days)
- [ ] RLS re-enable before prod
- [ ] iOS Safari compatibility — captureStream() unavailable, need fallback

---

## Long game (after product ships)
- [ ] ALIKE-N via onnxruntime-web — replace SIFT detector entirely
- [ ] WebGPU compute shaders — FAST corners + BRIEF in native GPU compute
- [ ] Fine-tune ALIKE on business card dataset — novel research contribution
