# SuperPoint & ALIKE — ML Feature Detectors for AR
Sources:
- https://github.com/rpautrat/SuperPoint
- https://github.com/magicleap/SuperPointPretrainedNetwork
- https://github.com/eric-yyjau/pytorch-superpoint
- https://github.com/Shiaoming/ALIKE
- Papers in knowledge/Superpoint_paper.pdf and knowledge/ALIKE_paper.pdf

## Why This Matters for BizzAR
ORB (what MindAR uses) fails on low-texture images. SuperPoint and ALIKE are ML-based detectors that find keypoints even where ORB finds nothing. Running one in the browser would make our AR engine dramatically better than any existing open-source WebAR tracker.

---

## SuperPoint

### What It Is
Self-supervised neural network that detects interest points (keypoints) AND computes descriptors from a single forward pass. Developed by Magic Leap (2018). Paper: "SuperPoint: Self-Supervised Interest Point Detection and Description" — DeTone, Malisiewicz, Rabinovich. arXiv:1712.07629.

### Architecture
- Fully convolutional VGG-style encoder (shared)
- Two decoder heads:
  1. Interest point detector (outputs keypoint heatmap)
  2. Descriptor generator (outputs 256-dim descriptors per keypoint)
- Input: grayscale image (any resolution — fully conv)
- Output: N keypoints + N descriptors

### Performance vs ORB (on HPatches benchmark)
| Metric | SuperPoint | ORB |
|---|---|---|
| Homography (illumination) | 0.965 | 0.523 |
| Homography (viewpoint) | 0.712 | 0.414 |

SuperPoint is ~1.8× better than ORB on both illumination and viewpoint changes.

### Training
1. Train MagicPoint on synthetic shapes (lines, polygons, etc.)
2. Homographic Adaptation: apply random homographies to real images (MS-COCO), run MagicPoint to get pseudo-labels
3. Train SuperPoint jointly (detector + descriptor) on these pseudo-labels
4. Repeat steps 2-3 to refine

Training time: ~8-10 hours on NVIDIA 2080Ti.

### Pretrained Weights
- `superpoint_v1.pth` — Magic Leap's official model (MIT license via pytorch-superpoint)
- `superPointNet_170000_checkpoint.pth.tar` — COCO-trained (pytorch-superpoint repo)
- Default input: 160×120px (configurable)

### License
- Magic Leap repo: Subject to Magic Leap Developer Agreement (restrictive)
- rpautrat/pytorch version: **MIT License** ✓ — this is the one to use

### Browser Deployment Path
No WASM port exists today. This is the gap we'd be filling.

To deploy in browser:
1. Convert PyTorch model to ONNX (`torch.onnx.export`)
2. Convert ONNX to TensorFlow.js or ONNX.js format
3. Run inference via WebGL backend (TFJS-WebGL or ort-web)
4. OR: Implement the architecture as native WebGL shaders (like WebAR.rocks does)

**Model size estimate:** VGG-style encoder is ~3-5MB compressed. Feasible for web delivery.

---

## ALIKE

### What It Is
"Accurate and Lightweight Keypoint Detection and Descriptor Extraction" (2022). Outperforms SuperPoint on multiple benchmarks while being significantly smaller. The right choice if we want to run ML detection in-browser.

Paper in: knowledge/ALIKE_paper.pdf

### Performance vs SuperPoint
| Metric | ALIKE-N | SuperPoint |
|---|---|---|
| MHA@3 on HPatches | 75.74% | 70.19% |
| mAA(10°) on IMW2020-test | 47.18% | 28.97% |

ALIKE-N is **significantly better** than SuperPoint on the IMW benchmark (real-world image matching).

### Model Sizes
| Variant | Parameters | GFLOPs (640×480) | Speed (Titan X) |
|---|---|---|---|
| ALIKE-N (tiny) | **318 KB** | 7.9 | 95 fps |
| ALIKE-L (larger) | 653 KB | 19.7 | — |

**318KB model weights** — extremely feasible for browser delivery. Compare to typical ML models (>50MB).

### Architecture Key Innovation
- Differentiable keypoint detection module — allows end-to-end training of both detector and descriptor
- Sub-pixel keypoint accuracy — detects at fractional pixel positions
- No complex multi-scale feature pyramid needed

### Why ALIKE > SuperPoint for Our Use Case
1. Smaller (318KB vs ~5MB) — better mobile load time
2. More accurate on real-world benchmarks
3. 95fps on desktop GPU — likely achievable at useful rates on mobile GPU via WebGL
4. Designed with lightweight deployment in mind

### Browser Deployment Path
Same as SuperPoint — needs ONNX conversion + WebGL backend. But the 318KB model size makes this far more practical.

### Training for Business Cards
To fine-tune ALIKE on business card images:
1. Collect ~1000 business card photos under varied lighting/angles
2. Use homographic adaptation (same as SuperPoint training) to generate pseudo-labels
3. Fine-tune ALIKE-N weights on this domain-specific data
4. Result: a detector specialized for business card keypoints

This would be a novel contribution — no ML-based keypoint detector has been fine-tuned for business card AR.

---

## Comparison Table for Our Decision

| Property | ORB (current) | SuperPoint | ALIKE-N |
|---|---|---|---|
| Type | Hand-crafted | ML | ML |
| Low-texture performance | Poor | Good | Better |
| Browser today | ✓ (built into MindAR) | ✗ (needs port) | ✗ (needs port) |
| Model size | N/A | ~5MB | 318KB |
| Speed (GPU) | Very fast | Fast | 95fps |
| License | Open | MIT | MIT |
| Fine-tuneable | ✗ | ✓ | ✓ |

## Implementation Plan (Long Game)
1. Convert ALIKE-N to ONNX
2. Run via onnxruntime-web with WebGL backend
3. Replace MindAR's ORB detector call with ALIKE output
4. Keep MindAR's pose estimation / One Euro filter pipeline
5. Fine-tune on business card dataset

This is a research-grade contribution. No open-source project has done this.
