# 03 — WebAR SLAM vs Image Tracking

**Date:** 2026-05-19
**Topic:** What SLAM means in a WebAR context, whether MindAR supports it, what the alternatives are post-8th-Wall, and how SLAM would improve BizzAR's tracking.

---

## Definitions

- **Image tracking** (a.k.a. natural feature tracking, marker tracking): every frame, the engine looks at the camera image, finds a known target image, solves its 6DOF pose. **No memory between frames** beyond optional smoothing. **If the camera looks away → pose is lost.**
- **SLAM** (Simultaneous Localization And Mapping): the engine builds a map of the surrounding environment from raw camera + IMU data, and continuously knows the camera's pose **relative to the world**, not relative to any one target. **Looking away does not lose the anchor.** This is what ARKit / ARCore / 8th Wall SLAM provide.
- **World tracking / Inside-out tracking**: marketing terms typically meaning SLAM with anchors.

---

## Does MindAR support SLAM?

**No.** MindAR is strictly an image tracking (and face tracking) library. From the documentation overview, there's no mention of world tracking or SLAM features. Confirmed by community statement: "MindAR.js doesn't support world tracking yet" ([Kivicube 2026 guide](https://www.kivicube.com/post/augmented-reality-after-8th-wall-shutdown-your-2026-guide-to-the-right-webar-platform/)).

Practical consequence: **the AR content is lost the instant the target image leaves the camera frame** (modulo `missTolerance`). For a business card use case where the user might rotate to inspect a 3D model, this is a hard ceiling on UX quality.

---

## SLAM alternatives in WebAR (2026 landscape)

### 8th Wall — now open source as of Feb 28, 2026

- 8th Wall's hosted commercial platform **shut down on 28 February 2026**.
- Code transitioned to open source at **8thwall.org**.
- **The Distributed Engine Binary, which includes SLAM, is available under a binary-only license for continued commercial and noncommercial use.** This means SLAM is *technically still available* — but binary-only, and the cloud/build/auth infrastructure is gone. Self-hosting required.
- Hosted campaigns viewable until Feb 28, 2027, then deleted.

Sources:
- [8th Wall open source announcement](https://www.8thwall.com/blog/post/208587408737/8th-wall-open-source)
- [AR Code timeline writeup](https://ar-code.com/blog/8th-wall-is-shutting-down-timeline-impact-and-the-best-8th-wall-alternative-for-webar)
- [Remix Reality coverage](https://www.remixreality.com/8th-wall-to-shut-down-after-seven-years-of-advancing-webar/)

### Mattercraft (Zappar)

- Uses **WebXR + App Clip** to delegate SLAM to native ARKit (iOS) / ARCore (Android) without an app download.
- Marketed as the closest "8th Wall replacement" with full world tracking.
- [Mattercraft for 8th Wall developers](https://zap.works/mattercraft-for-8th-wall-studio-developers/)

### Kivicube

- Pure-WebAR platform. Claims to support face AR, image AR, **SLAM experiences**. Closed-source, hosted.
- [Kivicube 2026 guide](https://www.kivicube.com/post/augmented-reality-after-8th-wall-shutdown-your-2026-guide-to-the-right-webar-platform/)

### Zapworks / HOVARLAY / ARLOOPA / AR Code / Wintor

- Various commercial WebAR platforms with mixed image + SLAM + face support. See [03-webar-slam-platforms.md] (none in this kit) or the source links.

### WebXR (browser-native)

- ARKit/ARCore exposed via the WebXR Device API in Chrome (Android), Edge, Samsung Internet, Oculus / Horizon Browser. **iOS Safari does not support WebXR** as of 2026 — this is the blocker for production WebAR on iPhone.
- See [07-webxr-image-tracking.md](./07-webxr-image-tracking.md) for the dedicated deep-dive.

---

## How SLAM would improve BizzAR specifically

1. **Persistent anchor after look-away.** User scans card, AR content appears, user rotates phone to inspect a 3D model from another angle → with image-tracking the content vanishes the moment the card leaves frame; with SLAM the content stays locked in space.
2. **Smoother pose during motion.** SLAM combines visual + IMU at 60+ Hz; image tracking is purely visual at ~30 fps and noisier.
3. **Larger working volume.** Image tracking needs the marker visible; SLAM lets you walk around the AR object.
4. **Better in low-light / motion blur.** IMU integration covers frames where vision fails.

---

## Practical hybrid: image-anchored SLAM

The state-of-the-art pattern (used by 8th Wall, Mattercraft):
1. User points phone at the marker — image tracking gives the **absolute scale and orientation** anchor.
2. Once anchored, switch primary tracking to SLAM. Image tracker can run intermittently to re-anchor / drift-correct.
3. Content stays in world space.

**This is not achievable purely in MindAR.** It would require:
- WebXR session (for SLAM via native ARCore/ARKit), or
- 8th Wall OSS binary (for proprietary SLAM), or
- A roll-your-own SLAM in WASM (very hard — ORB-SLAM3 ported via Emscripten is the only realistic option, and runs poorly on mobile).

---

## Actionable for BizzAR

1. **Accept the limitation for v1.** Document explicitly: "Keep the card visible while interacting." This is a known WebAR convention and users get it.
2. **Mitigate with `missTolerance` and IMU fusion:**
   - Crank `missTolerance` to ~30 so content lingers for ~1 second of look-away.
   - Hook into `DeviceOrientationEvent` (gyroscope) — when MindAR loses the target but the gyro shows the phone has only moved a small amount, freeze the last known pose for an extra ~500 ms before fading content out.
3. **Plan v2 path: WebXR on Android, MindAR fallback on iOS.**
   - Feature-detect `navigator.xr` + `immersive-ar` + `image-tracking` (where applicable).
   - On Android Chrome: use WebXR with image tracking module if available, else use WebXR hit-test + manual anchor placement, with MindAR as the discovery step.
   - On iOS Safari: stay on MindAR (no WebXR yet), document the limitation.
4. **Watch 8th Wall OSS.** The binary-only SLAM is the best WebAR-SLAM on the market. If BizzAR can ship a backend that bundles the 8thwall.org distributable, that's a moat — but it's a non-trivial integration and the binary license terms need careful review.
5. **Open-source contribution opportunity:** A clean "MindAR + WebXR fallback ladder" library that detects platform capability and degrades gracefully would be valuable to the WebAR community. (Flagged per BizzAR CLAUDE.md.)

---

## Sources

- [Kivicube 2026 alternatives guide](https://www.kivicube.com/post/augmented-reality-after-8th-wall-shutdown-your-2026-guide-to-the-right-webar-platform/)
- [Mattercraft for 8th Wall developers (Zappar)](https://zap.works/mattercraft-for-8th-wall-studio-developers/)
- [8th Wall open source announcement](https://www.8thwall.com/blog/post/208587408737/8th-wall-open-source)
- [AR Code 8th Wall shutdown timeline](https://ar-code.com/blog/8th-wall-is-shutting-down-timeline-impact-and-the-best-8th-wall-alternative-for-webar)
- [ARLOOPA migration guide](https://www.arloopa.com/blog/8th-wall-is-shutting-down-where-to-move-your-webar-projects)
- [Remix Reality on 8th Wall shutdown](https://www.remixreality.com/8th-wall-to-shut-down-after-seven-years-of-advancing-webar/)
- [Hololink WebAR editor comparison](https://www.hololink.io/blog-posts-hololink/choosing-the-right-webar-editor)
