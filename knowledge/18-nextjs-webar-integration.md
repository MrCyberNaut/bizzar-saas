# Next.js + WebAR Integration Patterns

**Date:** 2026-05-17
**Relevance:** bizzar/app/ar/[userId]/[cardId] route architecture

---

## 1. ssr: false Dynamic Import — Why Required

Three.js, MindAR, and react-three-mind all access `window`, `document`, `navigator`, and WebGL APIs at import time. SSR (Node.js) has none of these:

```typescript
// app/ar/ARPageClient.tsx
import dynamic from 'next/dynamic'

const ARScene = dynamic(
  () => import('@/components/ar/ARScene').then(m => ({ default: m.ARScene })),
  {
    ssr: false,           // REQUIRED — prevents Node.js import crash
    loading: () => <LoadingScreen />,
  }
)
```

**Bundle impact:** The entire AR bundle (~2MB: react-three-mind + MindAR + Three.js) is only loaded on the /ar route, never included in the main app bundle.

---

## 2. COOP/COEP Headers for WASM Threading

Required for `SharedArrayBuffer` (needed by MindAR WASM threaded mode):

```javascript
// next.config.js
const nextConfig = {
  async headers() {
    return [
      {
        // Scope ONLY to AR routes — global breaks Stripe, OAuth, Google Analytics iframes
        source: '/ar/:path*',
        headers: [
          { key: 'Cross-Origin-Opener-Policy',   value: 'same-origin' },
          { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
        ],
      },
    ]
  },
}
module.exports = nextConfig
```

**Warning:** Applying COOP/COEP globally breaks:
- Stripe Payment Element (cross-origin iframe)
- Google OAuth popup
- Any third-party `<iframe>` embed

Scope to `/ar/*` only.

---

## 3. Bundle Splitting Strategy

```typescript
// next.config.js — ensure AR vendors are split into their own chunk
const nextConfig = {
  webpack(config) {
    config.optimization.splitChunks = {
      ...config.optimization.splitChunks,
      cacheGroups: {
        ...config.optimization.splitChunks?.cacheGroups,
        arVendors: {
          test: /[\\/]node_modules[\\/](three|@react-three|react-three-mind|mind-ar)[\\/]/,
          name: 'ar-vendors',
          chunks: 'async',  // only load when AR route is visited
          priority: 20,
        },
      },
    }
    return config
  },
}
```

Result: AR bundle only loads when user navigates to `/ar/*`. Main app stays fast.

---

## 4. Service Worker — Cache .mind File

The `.mind` file (~500KB) is the MindAR compiled target. Cache it aggressively:

```javascript
// public/sw.js (or use next-pwa / workbox)
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('.mind')) {
    event.respondWith(
      caches.open('ar-targets-v1').then(cache =>
        cache.match(event.request).then(cached => {
          if (cached) return cached
          return fetch(event.request).then(response => {
            cache.put(event.request, response.clone())
            return response
          })
        })
      )
    )
  }
})
```

With `next-pwa`:
```javascript
// next.config.js
const withPWA = require('next-pwa')({ dest: 'public' })
module.exports = withPWA({
  // runtimeCaching for .mind files
})
```

---

## 5. Texture Loading — next/image vs Raw URL

```typescript
// ❌ Don't use next/image for AR textures
// next/image returns optimized srcset — useTexture needs a single URL

// ✅ Use raw public URL
const texture = useTexture('/card1.jpg')

// ✅ Or user-uploaded image from Supabase Storage
const cardImageUrl = supabase.storage.from('cards').getPublicUrl(card.image_path).data.publicUrl
const texture = useTexture(cardImageUrl)
```

**Gotcha:** `useTexture` fetches via XHR — the URL must be CORS-accessible. Supabase Storage is public by default ✅.

---

## 6. Environment Variables for AR Feature Flags

```bash
# .env.local
NEXT_PUBLIC_AR_ENABLE_SPARKLES=true
NEXT_PUBLIC_AR_ENABLE_3D_MODEL=false
NEXT_PUBLIC_AR_MAX_PANELS=1
```

```typescript
const sparklesEnabled = process.env.NEXT_PUBLIC_AR_ENABLE_SPARKLES === 'true'
```

Use for gradual feature rollout and A/B testing AR experiences.

---

## 7. Vercel Deployment Considerations

- **Serverless functions:** AR viewer `/ar/[userId]/[cardId]` is static — use `export const dynamic = 'force-static'` for ISR
- **CDN:** Static assets (`.mind`, card images) served from Vercel's CDN — fast globally
- **Edge:** Don't run AR logic at Edge — it's client-side only
- **Headers:** Vercel's `vercel.json` alternative for COOP/COEP on Vercel:

```json
{
  "headers": [
    {
      "source": "/ar/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }
  ]
}
```

---

## 8. Pre-warming MindAR on Idle

Eliminate the 500–1500ms init stall on first scan:

```typescript
// In the AR landing page (before user taps "Start AR")
useEffect(() => {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      // Pre-fetch .mind file into browser cache
      fetch('/card.mind', { priority: 'low' })
    })
  }
}, [])
```

---

## Actionable for BizzAR

1. **Now:** Add COOP/COEP headers scoped to `/ar/*` in next.config.js
2. **Now:** Add webpack splitChunks for ar-vendors chunk
3. **v1:** Service Worker for `.mind` file caching — critical for returning users
4. **v1:** `requestIdleCallback` pre-fetch of `.mind` on card share landing page
5. **v1:** `export const dynamic = 'force-static'` on AR viewer route for ISR

---

## Sources
- https://nextjs.org/docs/pages/building-your-application/optimizing/lazy-loading
- https://nextjs.org/docs/app/api-reference/next-config-js/headers
- https://web.dev/cross-origin-isolation-guide/
- https://vercel.com/docs/edge-network/headers
- https://developer.mozilla.org/en-US/docs/Web/API/Window/requestIdleCallback
