# getUserMedia Interception + Canvas captureStream — Technical Reference
Source: https://blog.mozilla.org/webrtc/enhancing-webcam-using-capturestream/

## The Core Pattern (We Have Implemented This)
```
getUserMedia() → raw camera stream
    ↓
Draw video frames to canvas
    ↓  
Apply WebGL shader effects (per-pixel GPU processing)
    ↓
canvas.captureStream() → enhanced MediaStream
    ↓
Return to calling code (MindAR) instead of raw stream
```

## Monkey-Patching getUserMedia
```javascript
const _original = navigator.mediaDevices.getUserMedia.bind(navigator.mediaDevices)

navigator.mediaDevices.getUserMedia = async (constraints) => {
  const rawStream = await _original(constraints)
  // ... process and return canvas stream
  return canvas.captureStream(60)
}
```

**Critical:** Must be installed before any code calls getUserMedia (timing matters).

## WebGL Shader Effect Types
All effects implemented as fullscreen quad fragment shaders:

1. **Color filters** — per-channel lookup tables (brightness, contrast, saturation)
2. **Distortion effects** — coordinate transformation (warp, fisheye, vignette)
3. **Composite overlays** — blending new pixels over existing frame

Effects can chain (output of one feeds next).

## Browser Compatibility Issues
Multiple getUserMedia variants exist:
- `navigator.mediaDevices.getUserMedia` (standard, modern)
- `navigator.getUserMedia` (legacy)
- `navigator.webkitGetUserMedia` (Chrome old)
- `navigator.mozGetUserMedia` (Firefox old)

**Our implementation:** Patches only `navigator.mediaDevices.getUserMedia`. This covers Chrome + Firefox + modern Safari. Legacy prefixed versions are not patched (not needed for 2026 targets).

## iOS Safari Limitation ⚠️
`HTMLCanvasElement.captureStream()` is NOT available in iOS Safari (as of 2026).
- This means our entire preprocessing pipeline silently fails on iPhone
- MindAR will fall back to... nothing. getUserMedia won't return a stream.
- **Fix needed:** Detect iOS Safari and skip the interceptor entirely

```javascript
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream
const isSafari = /^((?!chrome|android).)*safari/i.test(navigator.userAgent)
if (isIOS || isSafari) {
  // Do not install interceptor
}
```

## captureStream FPS Parameter
```javascript
canvas.captureStream(60) // request 60fps
```
- The 60 is a REQUEST not a guarantee
- Browser caps at display refresh rate (usually 60Hz, sometimes 120Hz on newer phones)
- On budget Android: actual delivery may be 30fps regardless of request
- On iOS: captureStream doesn't exist at all (see above)

## UNPACK_FLIP_Y_WEBGL
When uploading video frames to WebGL texture:
```javascript
gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true)
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, videoElement)
```
Without FLIP_Y=true, the video is rendered upside down (WebGL texture Y=0 is bottom, but video Y=0 is top).

## Performance Notes
- All shader processing happens on GPU — no CPU overhead for the effects themselves
- Main CPU cost: `texImage2D` call (uploading video frame to GPU) — runs every frame
- Target: <2ms per frame for the whole preprocessing pass on mid-range mobile
- If GPU is shared with MindAR's own WebGL context, there may be contention

## Our Implementation
Files: `bizzar-proto/src/lib/camera-preprocess.ts`

Key design decisions:
1. **Bypass uniform** (`u_bypass`) — toggle shader effects without stream restart
2. **Fire-and-forget play()** — don't await video.play() to avoid blocking getUserMedia return
3. **Program/uniforms set once** in constructor — not per-frame
4. **_boundLoop pattern** — bound method reference avoids closure allocation per rAF tick
5. **_original stored at module level** — start() must call _original, not navigator.mediaDevices.getUserMedia (infinite loop otherwise)
