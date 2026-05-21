# WebAR.rocks — Neural Network Object Tracking in Browser
Source: https://github.com/WebAR-rocks/WebAR.rocks.object

## What It Is
WebGL + JavaScript library for real-time object detection, tracking, and 6DoF pose estimation, entirely in-browser. Uses neural networks instead of ORB/SIFT feature matching.

## Why It Matters for BizzAR
This is proof that ML-based tracking runs at acceptable performance in browser WebGL today. The architecture is what we'd want to eventually implement for low-texture business cards.

## Core Architecture
- **No external ML frameworks** — no TensorFlow.js, no PyTorch. Pure WebGL compute.
- Neural network runs as WebGL shader programs (the GPU does inference)
- Trained on 3D synthetic data (not real photos) — object-specific models
- Detection → tracking pipeline (detect first, then track frame-to-frame)
- Async variant (`detect_async()`) uses WebGL2 async readback to reduce CPU blocking

## Performance
- Input resolution: 64×64 to 128×128 pixels (very small — intentional for speed)
- Recommended: 3–6 detections per loop call
- Models are GZIP-compressed, hosted on CDN
- No published FPS numbers but runs real-time on mobile based on demos
- WebGL2 preferred; WebGL1 fallback with OES_TEXTURE_FLOAT extension

## API Pattern
```javascript
WebARRocksObject.init({
  canvas: canvasElement,
  video: videoElement,
  NNs: [{ path: 'model_url.json', label: 'myObject' }],
  callback: (err) => { /* ready */ }
})

// Per-frame detection
const result = WebARRocksObject.detect(nDetectionsPerLoop)
// result: { label, positionX, positionY, scale, yaw, confidence }
```

## Output Format
```
{
  label: 'myObject',
  positionX: 0.0-1.0,   // normalized 2D position
  positionY: 0.0-1.0,
  scale: float,          // scale factor
  yaw: float,            // rotation around Y axis
  confidence: 0.0-1.0    // detection confidence
}
```
Note: Only yaw angle (1 rotation DOF), not full 6DoF rotation. For full 6DoF you need their `WebAR.rocks.train`.

## Key Difference From ORB-Based Tracking
| Property | ORB (MindAR) | WebAR.rocks Neural |
|---|---|---|
| Feature extraction | Hand-crafted (corners/edges) | Learned representation |
| Low texture handling | Poor | Better (learned from 3D) |
| Requires | Feature-rich target image | Custom trained model |
| Setup | Compile .mind from any photo | Train/find model for your object |
| Browser support | All browsers (JS only) | WebGL required |
| Speed | Fast | Fast (GPU-accelerated) |

## Limitations
- Requires custom model training per object class
- Not plug-and-play for arbitrary user-uploaded images (unlike MindAR)
- Highly reflective/refractive objects are "harder to detect"
- No partial visibility (object must fit in frame)
- Only 1 rotation DOF in standard version (yaw only)
- Max 3 simultaneous object types, 1 tracked at a time

## Relevance to BizzAR
**Direct use:** Unlikely. Requires per-card model training — not scalable for a SaaS where users upload arbitrary cards.

**Architecture inspiration:** The WebGL-native inference approach is exactly what a browser-port of ALIKE or SuperPoint should look like. No TF.js overhead. Pure WebGL shader compute. This is the target architecture for our long-game ML detector.

**Training approach inspiration:** Their synthetic 3D training data approach could work for business cards — generate synthetic card images with known keypoints, train a lightweight detector to find them.
