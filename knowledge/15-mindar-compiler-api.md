# MindAR Compiler API — Deep Research

## Overview

MindAR's Compiler is a pure JavaScript (WebGL-accelerated) tool that runs entirely in the browser. No server round-trip is needed. It converts raw images into `.mind` files — compact binary feature data that the runtime uses to detect and track targets. Because it runs in-browser, you can integrate it directly into a Next.js card editor, letting users preview trackability before saving.

---

## How the Compiler Works (Architecture)

The Compiler lives at `src/image-target/compiler.js` in the mind-ar-js repo. It extends `CompilerBase` and spawns a **Web Worker** to do the heavy lifting off the main thread.

Pipeline inside the worker:
1. Convert each input image to **grayscale** (RGB averaging)
2. Build an **image pyramid** — multiple downscaled copies for scale-invariant feature detection
3. Detect **keypoints** at each scale level using a custom TF.js WebGL shader (scale-space extrema detection similar to DoG/FAST)
4. Compute **BRIEF-style descriptors** for each keypoint — binary patch comparisons stored as bit vectors
5. Output two data sets per image:
   - `matchingData` — KPM (keypoint matching) feature set: `maximaPoints` + `minimaPoints`
   - `trackingData` — lighter point set used for frame-to-frame tracking

Compilation runs at **50% progress each** for matching and tracking phases.

---

## The Compiler Class API

### Import

```js
// From npm package (use in browser bundle / Next.js with SSR disabled)
import { Compiler } from 'mind-ar/dist/mindar-image.prod.js';

// From source (dev/examples)
import { Compiler } from '../../src/image-target/compiler.js';
```

> **Next.js caveat:** `mind-ar` uses Web Workers and Canvas APIs. Always import with `ssr: false` — see Next.js Integration section below.

### Instantiation

```js
const compiler = new Compiler();
```

No constructor arguments.

### `compileImageTargets(images, progressCallback)`

```js
const dataList = await compiler.compileImageTargets(images, (progress) => {
  console.log(`Compiling: ${progress.toFixed(2)}%`);
});
```

| Parameter | Type | Description |
|---|---|---|
| `images` | `HTMLImageElement[]` | Array of loaded `<img>` elements |
| `progressCallback` | `(percent: number) => void` | Called with 0–100 as compilation proceeds |

**Returns:** `Promise<DataList>` — the compiled feature data (see structure below).

### `exportData()`

```js
const buffer = await compiler.exportData();
// buffer is an ArrayBuffer — the binary .mind file
```

Returns a **msgpack-encoded binary buffer** prefixed with a version byte (`CURRENT_VERSION = 2`). This is what you upload to Supabase Storage.

### `importData(buffer)`

```js
// Load an existing .mind file back into the compiler
const dataList = compiler.importData(arrayBuffer);
```

Decodes the msgpack buffer and reconstructs the data array. Validates version compatibility.

---

## DataList Structure

```ts
// What compileImageTargets returns
interface DataList {
  targetImages: HTMLImageElement[];   // original inputs
  imageList: ProcessedImage[][];      // pyramid levels per image
  matchingData: MatchingData[];       // one entry per target image
  trackingData: TrackingData[];       // one entry per target image
}

interface MatchingData {
  maximaPoints: KeyPoint[];   // local maxima keypoints
  minimaPoints: KeyPoint[];   // local minima keypoints
  // at multiple pyramid scales
}

interface KeyPoint {
  x: number;
  y: number;
  scale: number;
  // binary descriptor bits (BRIEF-style)
}

interface TrackingData {
  points: { x: number; y: number }[];  // tracking feature coordinates
}
```

Each returned `dataList[i]` contains:
- `trackingImageList` — processed target images
- `trackingData[i].points` — tracking feature coordinates `{x, y}`
- `matchingData[i].maximaPoints` and `matchingData[i].minimaPoints` — KPM features

---

## Extracting a Feature Count (Trackability Score)

```js
const dataList = await compiler.compileImageTargets([img], (p) => {});

// For target index 0:
const trackingCount = dataList[0].trackingData[0].points.length;
const kpmCount =
  dataList[0].matchingData[0].maximaPoints.length +
  dataList[0].matchingData[0].minimaPoints.length;

console.log(`Tracking features: ${trackingCount}`);
console.log(`KPM features:      ${kpmCount}`);
```

**Interpretation:**
- `kpmCount` is the primary quality signal — these are the features used for initial detection
- `trackingCount` matters for how stable the lock stays frame-to-frame
- Community threshold: **`kpmCount > 300` is considered a good target**; under 100 is very poor

---

## Visualizing the Feature Heatmap

From the official `compile.html` example, features are rendered as **green dot clusters** on a canvas overlay:

```js
function drawFeaturePoints(canvas, img, dataItem) {
  const ctx = canvas.getContext('2d');
  canvas.width = img.width;
  canvas.height = img.height;

  // Draw the original image as background
  ctx.drawImage(img, 0, 0);

  // Get pixel data for overlay
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const data = imageData.data;

  // Green pixel color in ABGR packed int
  const pix = (0xff << 24) | (0x00 << 16) | (0xff << 8) | 0x00;

  // Draw each feature point as a 6-pixel cross
  const allPoints = [
    ...dataItem.matchingData[0].maximaPoints,
    ...dataItem.matchingData[0].minimaPoints,
  ];

  allPoints.forEach(({ x, y }) => {
    const cx = Math.round(x);
    const cy = Math.round(y);
    for (let dx = -6; dx <= 6; dx++) {
      for (let dy = -6; dy <= 6; dy++) {
        if (Math.abs(dx) + Math.abs(dy) < 4) {  // cross shape
          const px = cx + dx;
          const py = cy + dy;
          if (px >= 0 && px < canvas.width && py >= 0 && py < canvas.height) {
            const offset = (py * canvas.width + px) * 4;
            data[offset]     = 0;    // R
            data[offset + 1] = 255;  // G
            data[offset + 2] = 0;    // B
            data[offset + 3] = 255;  // A
          }
        }
      }
    }
  });

  ctx.putImageData(imageData, 0, 0);
}
```

A sparse visualization (features clustered in one corner, large blank areas) signals a poor target.

---

## Defining a Quality Threshold

Build a simple scoring function from the extracted data:

```ts
interface TrackabilityResult {
  kpmFeatureCount: number;
  trackingFeatureCount: number;
  score: 'excellent' | 'good' | 'fair' | 'poor';
  message: string;
}

function assessTrackability(dataList: any[], targetIndex = 0): TrackabilityResult {
  const kpmCount =
    dataList[targetIndex].matchingData[0].maximaPoints.length +
    dataList[targetIndex].matchingData[0].minimaPoints.length;
  const trackingCount = dataList[targetIndex].trackingData[0].points.length;

  let score: TrackabilityResult['score'];
  let message: string;

  if (kpmCount >= 500) {
    score = 'excellent';
    message = 'Great target — fast lock-on, stable tracking.';
  } else if (kpmCount >= 300) {
    score = 'good';
    message = 'Good target — reliable in normal lighting.';
  } else if (kpmCount >= 100) {
    score = 'fair';
    message = 'Mediocre target — may struggle in poor lighting.';
  } else {
    score = 'poor';
    message = 'Bad target — likely to fail detection. Add more visual detail.';
  }

  return { kpmFeatureCount: kpmCount, trackingFeatureCount: trackingCount, score, message };
}
```

---

## Next.js Integration — Full UX Flow

### Step 1: Dynamic import (mandatory — avoids SSR crash)

```tsx
// app/editor/components/TrackabilityChecker.tsx
'use client';
import dynamic from 'next/dynamic';

const TrackabilityChecker = dynamic(
  () => import('./TrackabilityCheckerInner'),
  { ssr: false, loading: () => <p>Loading analyzer...</p> }
);

export default TrackabilityChecker;
```

### Step 2: The actual compiler component

```tsx
// TrackabilityCheckerInner.tsx — loaded only on client
'use client';
import { useState, useRef } from 'react';

export default function TrackabilityCheckerInner({ imageUrl }: { imageUrl: string }) {
  const [status, setStatus] = useState<'idle' | 'compiling' | 'done'>('idle');
  const [progress, setProgress] = useState(0);
  const [result, setResult] = useState<TrackabilityResult | null>(null);
  const canvasRef = useRef<HTMLCanvasElement>(null);

  async function analyzeImage() {
    setStatus('compiling');
    setProgress(0);

    // Dynamic import — only runs in browser
    const { Compiler } = await import('mind-ar/dist/mindar-image.prod.js');
    const compiler = new Compiler();

    // Load image as HTMLImageElement
    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.src = imageUrl;
    await new Promise(r => { img.onload = r; });

    // Compile
    const dataList = await compiler.compileImageTargets([img], (p: number) => {
      setProgress(Math.round(p));
    });

    // Score it
    const assessment = assessTrackability(dataList);
    setResult(assessment);

    // Draw feature overlay
    if (canvasRef.current) {
      drawFeaturePoints(canvasRef.current, img, dataList[0]);
    }

    setStatus('done');
  }

  return (
    <div className="space-y-4">
      <button onClick={analyzeImage} disabled={status === 'compiling'}>
        {status === 'compiling' ? `Analyzing... ${progress}%` : 'Check Trackability'}
      </button>

      {result && (
        <div className={`p-3 rounded ${
          result.score === 'excellent' ? 'bg-green-100' :
          result.score === 'good' ? 'bg-blue-100' :
          result.score === 'fair' ? 'bg-yellow-100' : 'bg-red-100'
        }`}>
          <p className="font-semibold capitalize">{result.score}</p>
          <p>{result.message}</p>
          <p className="text-sm text-gray-600">
            {result.kpmFeatureCount} detection features · {result.trackingFeatureCount} tracking features
          </p>
        </div>
      )}

      <canvas ref={canvasRef} className="w-full rounded border" />
    </div>
  );
}
```

### Step 3: Webpack config (if bundler complains about Web Workers)

In `next.config.js` or `next.config.ts`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  webpack: (config) => {
    // mind-ar uses ?worker&inline — ensure webpack handles it
    config.module.rules.push({
      test: /\.worker\.(js|ts)$/,
      use: { loader: 'worker-loader', options: { inline: 'no-fallback' } },
    });
    return config;
  },
};
export default nextConfig;
```

> In practice, the `mindar-image.prod.js` bundle inlines the worker, so this rule is often not needed. Test first.

---

## Exporting and Uploading the .mind File

After compilation, export and upload to Supabase Storage:

```ts
async function compileAndUpload(imageFile: File, cardId: string) {
  const { Compiler } = await import('mind-ar/dist/mindar-image.prod.js');
  const compiler = new Compiler();

  const img = new Image();
  const url = URL.createObjectURL(imageFile);
  img.src = url;
  await new Promise(r => { img.onload = r; });

  await compiler.compileImageTargets([img], () => {});
  const buffer = await compiler.exportData();

  // Upload ArrayBuffer to Supabase Storage
  const { error } = await supabase.storage
    .from('mind-files')
    .upload(`cards/${cardId}/target.mind`, buffer, {
      contentType: 'application/octet-stream',
      upsert: true,
    });

  URL.revokeObjectURL(url);
  return error;
}
```

---

## The .mind File Format

Binary structure (from `compiler-base.js`):
- **Prefix:** 1-byte version number (`CURRENT_VERSION = 2`)
- **Body:** msgpack-encoded object:

```
{
  v: 2,
  dataList: [
    {
      targetImage: { width: number, height: number },
      trackingData: [...],   // tracking feature points
      matchingData: [...]    // KPM keyframes at multiple scales
    },
    // ... one entry per compiled image
  ]
}
```

Raw image pixel data is NOT stored in the .mind file (only dimensions). This keeps file sizes small — typically 50–200 KB per target.

---

## Command-Line Compilation

As of 2026, **no official CLI tool exists**. This is an open feature request (Discussion #536 on the repo — still unanswered since March 2026). The only option for automated/server-side compilation is either:
- A headless browser (Puppeteer/Playwright) running the browser Compiler API
- A third-party package like `@maherboughdiri/mind-ar-compiler` (thin wrapper around the same browser API)
- The Pictarize web tool at https://pictarize.com/image-analyzer/

---

## Actionable for BizzAR

| Integration Point | Recommendation |
|---|---|
| Card editor upload flow | Run `compileImageTargets` client-side immediately after card image upload |
| Trackability gate | Block AR activation if `kpmCount < 100`; warn at `< 300` |
| Feature heatmap | Show canvas overlay with green dots so users see sparse areas |
| .mind file storage | Upload `exportData()` buffer to Supabase Storage `mind-files` bucket |
| Progress UX | Show `%` progress bar — compilation takes 2–10s depending on image size |
| SSR | Always `dynamic(() => import(...), { ssr: false })` for anything using `Compiler` |
| Recompile trigger | Recompile whenever the user changes the card's background image |

---

## Sources

- [MindAR Core API Docs](https://hiukim.github.io/mind-ar-js-doc/core-api/)
- [Compile Target Images Docs](https://hiukim.github.io/mind-ar-js-doc/quick-start/compile/)
- [Image Targets Compiler Tool](https://hiukim.github.io/mind-ar-js-doc/tools/compile/)
- [compile.html example](https://github.com/hiukim/mind-ar-js/blob/master/examples/image-tracking/compile.html)
- [hiukim/mind-ar-js GitHub](https://github.com/hiukim/mind-ar-js)
- [Compiler class source](https://github.com/hiukim/mind-ar-js/blob/master/src/image-target/compiler.js)
- [CompilerBase source](https://github.com/hiukim/mind-ar-js/blob/master/src/image-target/compiler-base.js)
- [Discussion #536 — CLI tool request](https://github.com/hiukim/mind-ar-js/discussions/536)
- [Issue #163 — Compiler API question](https://github.com/hiukim/mind-ar-js/issues/163)
- [Pictarize image analyzer](https://pictarize.com/image-analyzer/)
- [How to Choose Part 4 — Pictarize scoring](https://www.mindar.org/how-to-choose-a-good-target-image-for-tracking-in-ar-part-4/)
