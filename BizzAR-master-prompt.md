# BizzAR — Master Context Prompt

Use this document to onboard any AI assistant, developer, or collaborator onto the BizzAR project with full context. Paste this at the start of any new session.

---

## What is BizzAR?

BizzAR is a WebAR SaaS product that transforms physical business cards into interactive augmented reality experiences. The name is a portmanteau of "Biz" (business) and "AR" (augmented reality).

A professional signs up, uploads a photo of their physical business card, and builds a set of floating interactive panels around it using a drag-and-drop editor. They publish their experience and get a shareable URL + QR code to print on their physical card. When someone receives that card, they point their phone camera at it — the browser opens automatically, no app install required — and the panels appear floating around the card in 3D space. They can tap modules to download a contact card, follow social links, watch a video, book a meeting, and more.

**Tagline:** Business cards with AR — point, scan, connect.

---

## Target User

Individual professionals: freelancers, consultants, founders. Single-user accounts at MVP. No team/enterprise features in scope.

---

## Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Framework | Next.js 14 (App Router, TypeScript) | Existing expertise, SSR + static routes, API routes |
| Styling | Tailwind CSS | Utility-first, consistent with existing projects |
| 3D Hub Editor | React Three Fiber (R3F) + Drei | React-native Three.js; Drei gives orbit controls, drag handles, HTML-in-3D for free |
| AR Runtime | 8th Wall OSS Image Targets + A-Frame | 8th Wall transitioned to open source (MIT) in Feb 2026; best-in-class image tracking, free for commercial use; works with A-Frame and Three.js |
| UI Animation | Framer Motion | UI transitions, drawer animations |
| Backend | Supabase (Postgres + Auth + Storage + Edge Functions + Realtime) | Auth, DB, file storage, serverless functions, realtime plan sync |
| Payments | Stripe | Subscription billing, webhook for plan upgrades |
| Image Target Compilation | 8th Wall `image-target-cli` | Compiles uploaded card image into a `.mind` binary file used by the AR runtime for tracking |
| QR Code | qrcode.js | Client-side QR generation, no server needed |

### Critical Architectural Note — Two Separate Runtimes

The project has two completely separate rendering environments:

1. **The editor** (`/editor/[cardId]`) — a standard Next.js + React page with an R3F 3D canvas for the hub view and a React drawer for the 2D panel editor. Full React component tree, Tailwind, Framer Motion, Supabase client.

2. **The AR viewer** (`/ar/[userId]/[cardId]`) — a **standalone HTML page with zero React and zero Next.js shell**. 8th Wall and A-Frame are loaded via plain `<script>` tags. This is intentional and non-negotiable: AR runtimes conflict with React's DOM reconciliation, and the bundle must be tiny for fast mobile load. This page fetches its card config from Supabase REST directly (no SDK), boots the AR runtime, and renders A-Frame entities.

These two environments must never be mixed.

---

## Application Routes

| Route | Type | Description |
|---|---|---|
| `/` | Next.js page | Marketing landing page |
| `/pricing` | Next.js page | Plan comparison, Stripe Checkout CTA |
| `/dashboard` | Next.js page (auth-gated) | Card list, create new card button |
| `/dashboard/cards` | Next.js page (auth-gated) | Card management |
| `/editor/[cardId]` | Next.js page (auth-gated) | 3D Hub + 2D panel editor |
| `/ar/[userId]/[cardId]` | Standalone HTML (no React) | Live AR experience, publicly accessible |

Publishing is **path-based, not subdomain-based** at MVP. Vanity subdomains (`user.bizzar.app`) are a future Pro feature. This eliminates all subdomain hosting cost complexity at launch.

---

## Database Schema (Supabase / Postgres)

### `users`
Extended from `auth.users`. Created via Supabase trigger on signup.

```sql
id              uuid PRIMARY KEY  -- mirrors auth.users.id
email           text
plan            text DEFAULT 'free'  -- 'free' | 'pro'
stripe_customer_id  text            -- nullable
created_at      timestamptz DEFAULT now()
```

### `cards`
Root entity. One row per business card.

```sql
id                  uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id             uuid REFERENCES users(id) ON DELETE CASCADE
name                text                        -- user-given label
card_image_url      text                        -- Supabase Storage path (original upload)
image_target_url    text                        -- Supabase Storage path (.mind compiled file)
published           boolean DEFAULT false
created_at          timestamptz DEFAULT now()
```

On card creation, a Postgres trigger (or app-level logic) automatically inserts 4 rows into `panels` — one per slot.

### `panels`
4 panels per card, auto-created. Fixed slot positions.

```sql
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
card_id     uuid REFERENCES cards(id) ON DELETE CASCADE
slot        text    -- 'overlay' | 'top' | 'left' | 'right'
enabled     boolean DEFAULT false   -- set true for overlay+top on free, all 4 on pro
grid_cols   int2 DEFAULT 4
grid_rows   int2 DEFAULT 4
```

### `modules`
Each module placed on a panel grid by the user.

```sql
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
panel_id    uuid REFERENCES panels(id) ON DELETE CASCADE
type        text    -- see Module Types below
grid_x      int2    -- top-left cell column (0-indexed)
grid_y      int2    -- top-left cell row (0-indexed)
grid_w      int2    -- width in cells
grid_h      int2    -- height in cells
config      jsonb   -- module-specific data
```

### RLS Policies
- All tables: users can only read/write their own rows (auth.uid() = user_id).
- `cards` and their nested `panels`/`modules`: public read allowed only when `cards.published = true` — this powers the AR viewer without requiring auth.
- `users.plan` is the single source of truth for freemium gating. The gate is never enforced client-side only.

---

## 3D Panel Layout (Spatial Model)

The editor shows the business card in an isometric top-slant view (similar to Blender's default perspective). Four panels float around it at fixed spatial positions:

| Slot | 3D Position | Description |
|---|---|---|
| `overlay` | Directly on top of card surface | Sits flat on the card, same plane. Used for the animated video overlay module that makes the card "come alive". |
| `top` | Above and slightly in front, 45° tilt | Like a Nintendo DS top screen — tilted toward the viewer, floating above the card. This is the most prominent panel. |
| `left` | Left of the `top` panel, angled inward | Wing panel. Visible from the viewer's perspective. |
| `right` | Right of the `top` panel, angled inward | Wing panel. Visible from the viewer's perspective. |

**Free plan:** `overlay` + `top` panels only. `left` and `right` are visible in the editor but rendered as dimmed, locked planes with a Pro badge.

**Pro plan:** All 4 panels active.

In the AR viewer, panels are `<a-entity>` elements parented to the 8th Wall image target anchor. Their 3D transforms (position, rotation) are pre-calculated constants that match the editor layout exactly — no runtime calculation needed.

---

## Panel Grid System

Each panel uses a **4-column × 4-row magnetic grid**. This is the same mental model as the iOS home screen widget system.

### How it works
- The grid is 4×4 cells. Each cell is a fixed size.
- Modules snap to grid cells on drop.
- Modules occupy a rectangular region: `grid_x`, `grid_y`, `grid_w`, `grid_h`.
- No two modules can overlap — collision detection on every drop.
- If a dropped module's target cells are occupied, it bounces back to the tray.
- Placed modules can be tapped to switch size variant. If the new size doesn't fit, user is prompted.
- Every change autosaves to Supabase immediately.

### Module Size Variants
Modules do **not** support free resize. Each module type has a defined set of valid size presets. The user picks from this list. The DB stores the raw `grid_w`/`grid_h` integers — size variant names (e.g. "2×2") are UI labels only.

---

## Module Catalogue

### Free Modules

| Module | Type Key | Min Size | Available Sizes | Description |
|---|---|---|---|---|
| Add to Contacts | `vcard` | 1×1 | 1×1, 2×1, 4×1 | Tap to download a `.vcf` contact file. Name, email, phone, company, website from config. |
| Social Icons | `socials` | 1×1 | 1×1, 2×1, 4×1 | Row of tappable icons. LinkedIn, Twitter/X, Instagram, GitHub, personal site. |
| Bio / Text Block | `bio` | 1×1 | 1×1, 2×1, 2×2, 4×1, 4×2 | Short tagline or bio. Plain text. |

### Pro Modules

| Module | Type Key | Min Size | Available Sizes | Description |
|---|---|---|---|---|
| GitHub Graph | `github` | 2×1 | 2×1, 4×1, 4×2 | Live GitHub contribution graph fetched from GitHub API. Username stored in config. |
| Image Gallery | `image_gallery` | 1×1 | 1×1, 2×1, 2×2, 4×2, 4×4 | Up to 4 images. Tap to expand. Uploaded to Supabase Storage. |
| Video Panel | `video` | 1×1 | 1×1, 2×2, 2×3, 4×2, 4×4 | Autoplay looping video. Upload mp4 or paste YouTube/Vimeo URL. |
| Animated Overlay | `animated_overlay` | fills panel | fills `overlay` panel only | Plays mp4 or GIF directly on the card surface in AR. Makes the card "come alive". Special-cased — ignores grid, fills entire overlay panel. |
| Calendly | `calendly` | 1×1 | 1×1, 2×1, 4×1, 4×2 | Tap to open Calendly booking URL in browser. Stores link only. |

### Module `config` JSONB Schema (per type)

```json
vcard:            { "name": "", "email": "", "phone": "", "company": "", "website": "" }
socials:          { "links": [{ "platform": "linkedin", "url": "" }] }
bio:              { "text": "" }
github:           { "username": "" }
image_gallery:    { "images": [{ "url": "", "alt": "" }] }
video:            { "url": "", "type": "upload | youtube | vimeo" }
animated_overlay: { "url": "", "type": "mp4 | gif" }
calendly:         { "url": "" }
```

---

## Freemium Gate

| Feature | Free | Pro |
|---|---|---|
| Active panels | overlay + top | All 4 |
| Free modules | vCard, Socials, Bio | vCard, Socials, Bio |
| Pro modules | Locked (greyed out in tray) | All 8 |
| Cards | 1 card max | Unlimited |
| Watermark in AR | "Made with BizzAR" badge | Removed |
| Vanity URL | No | Coming later |

**Enforcement:** `users.plan` is the server-side gate. RLS on `panels` prevents free users from saving modules to locked panels. Locked panels return `enabled = false` from the DB — the UI reflects this but does not enforce it alone.

**Upgrade flow:** Stripe Checkout → `checkout.session.completed` webhook → Supabase Edge Function updates `users.plan = 'pro'` → Supabase Realtime pushes the change to the open editor session → locked panels unlock instantly with no page reload.

**Watermark:** Rendered as a fixed A-Frame HTML overlay in the AR scene. Small, tasteful. Every shared AR experience is organic marketing.

---

## Creator User Flow (Step by Step)

### Step 1 — Sign Up
- Route: `/`
- Auth: Supabase magic link or Google OAuth
- On success: redirect to `/dashboard`
- `users` row created via Supabase trigger. `plan = 'free'`.

### Step 2 — Create a Card
- Route: `/dashboard`
- User clicks "New Card", gives it a name, uploads front-of-card image.
- On upload: Supabase Storage saves the image. Edge Function fires:
  - Runs `8th Wall image-target-cli` on the image.
  - Compiles a `.mind` binary file (the AR image target).
  - Saves `.mind` file back to Supabase Storage.
  - Updates `cards.image_target_url`.
- 4 `panels` rows auto-created: `overlay` and `top` with `enabled = true`, `left` and `right` with `enabled = false` (for free plan).
- User is redirected to `/editor/[cardId]`.

### Step 3 — Hub Editor (3D View)
- Route: `/editor/[cardId]`
- R3F canvas renders business card mesh (textured with uploaded card image) in isometric view.
- 4 panel planes float at their fixed spatial positions around the card.
- OrbitControls (Drei) — user can rotate view to inspect layout.
- Free panels (overlay, top): full opacity, clickable.
- Locked panels (left, right): dimmed, Pro badge, click opens upgrade prompt.

### Step 4 — Panel Editor (2D Grid)
- Clicking an active panel opens a drawer/modal with the 2D grid editor.
- Left tray: module catalogue. Pro modules shown with lock badge for free users.
- Drag module from tray → drop onto grid → size picker appears → user picks variant → module snaps into place.
- Click a placed module → open config form (fill in links, upload media, etc.).
- All changes autosave to Supabase (`modules` table upsert).
- User can remove modules (returns cell space to grid).

### Step 5 — Publish
- User clicks "Publish" in editor header.
- Sets `cards.published = true`.
- Dashboard shows:
  - Shareable URL: `bizzar.app/ar/[userId]/[cardId]`
  - QR Code: generated client-side via `qrcode.js`, downloadable as PNG.
- User prints QR on physical card or shares URL digitally.

### Step 6 — Upgrade to Pro (Optional)
- User clicks "Upgrade" anywhere a locked feature is shown.
- Stripe Checkout opens (hosted page).
- On payment success: `checkout.session.completed` webhook fires → Edge Function sets `users.plan = 'pro'` → Supabase Realtime notifies the open editor → left and right panels unlock in real time.

---

## Recipient User Flow (AR Experience)

1. Recipient receives physical business card. QR code is printed on it.
2. Scans QR with phone camera → browser opens `bizzar.app/ar/[userId]/[cardId]`.
3. Page loads (standalone HTML, no React):
   - Fetches `cards` row (verifies `published = true`).
   - Fetches all enabled `panels` + their `modules` with config.
   - Fetches `.mind` file URL.
   - Loads 8th Wall Image Targets + A-Frame via script tags.
   - Requests camera permission.
4. 8th Wall registers `.mind` file as the image target. Scans camera feed continuously.
5. Card detected in frame:
   - A-Frame spawns `<a-entity>` panel containers at pre-defined 3D transforms relative to the card anchor.
   - Each panel renders its modules — interactive modules as HTML overlays, media as A-Frame planes.
   - Animated overlay (if present) plays video/GIF on the card surface plane.
6. Recipient interactions:
   - Tap vCard module → `.vcf` file downloads → opens in Contacts app.
   - Tap social icon → link opens in browser.
   - Tap Calendly → booking page opens in browser.
   - Video modules autoplay looping.
   - Image gallery tap → expands image.
7. Free plan: "Made with BizzAR" watermark badge visible in corner of AR scene.

---

## 8th Wall OSS — What Changed & Why We're Using It

8th Wall was previously a paid WebAR platform (~$200/month commercial). In February 2026, Niantic completed its transition to open source at [8thwall.org](https://8thwall.org).

- **Image Targets module** — MIT licensed. This is what we use. It handles all image tracking (point camera at card → detect card → anchor AR content).
- **XR Engine (SLAM)** — distributed as a free closed-source binary under a limited-use commercial license. We do NOT need SLAM for BizzAR (SLAM is for surface/world tracking). Image Targets is sufficient.
- **image-target-cli** — open source tool that compiles a reference image (our business card photo) into a `.mind` binary. This runs in our Supabase Edge Function on card upload.
- Works natively with A-Frame and Three.js.
- No login, no hosting dependency, self-hosted entirely.

---

## Supabase Edge Functions

### `compile-image-target`
- **Trigger:** Called after card image is uploaded to Storage.
- **What it does:** Downloads the image, runs `image-target-cli`, uploads the resulting `.mind` file back to Storage, updates `cards.image_target_url`.
- **Note:** `image-target-cli` is a Node.js CLI tool — confirm it runs in Deno (Supabase Edge runtime) or wrap in a small Docker/Node function if needed.

### `stripe-webhook`
- **Trigger:** Stripe `checkout.session.completed` event.
- **What it does:** Verifies webhook signature, reads `customer_id`, looks up user, sets `users.plan = 'pro'`, updates `panels.enabled = true` for `left` and `right` panels on all the user's cards.

---

## Key Design Decisions & Rationale

| Decision | Rationale |
|---|---|
| AR viewer is standalone HTML, not React | AR runtimes (8th Wall + A-Frame) conflict with React DOM. Bundle must be tiny for mobile. |
| Path-based publish URLs, not subdomains | Subdomains require wildcard DNS + SSL cert management. Path routing is free, instant, and simpler at MVP. |
| Freemium gate server-side via RLS | Client-side gating is trivially bypassable. All enforcement at DB layer. |
| 4×4 preset grid, no free resize | Predictable layouts that translate cleanly into 3D AR space. Free resize creates layout chaos. |
| Supabase Realtime for plan upgrades | Feels instant and polished — locked panels unlock without page reload after Stripe payment. |
| Animated overlay ignores grid | It renders on the card surface itself, not on a floating panel. Different semantic from grid modules. |
| 8th Wall OSS over MindAR | Better tracking quality, enterprise-proven, now free and MIT for Image Targets. |
| R3F over raw Three.js | State management, drag handles, orbit controls via Drei — all native to React. Less imperative glue code. |

---

## Out of Scope for MVP

- Native app wrapper (pure WebAR only)
- SLAM / surface tracking (Image Targets only)
- Team / multi-user accounts
- Analytics (scan counts, tap rates)
- Custom domains or vanity subdomains
- A/B panel variants
- Backend/server-side module rendering
- Offline mode
- Social feed / public profile discovery

---

## Open Questions (Decide Before Implementation)

1. **Re-publish UX:** Should edits go live instantly on save, or require an explicit "Publish" button press? Recommendation: explicit publish — safer, gives user control, prevents half-finished edits going live.

2. **Image target quality warning:** 8th Wall CLI outputs a quality score for the compiled target. Should we surface a warning in the upload UI if the card is too low-contrast for reliable tracking? Recommendation: yes — prevents bad UX downstream.

3. **Video upload size cap:** Pick a hard limit (recommendation: 20MB) before building the upload UI. Prevents runaway Supabase Storage costs on the free tier.

---

## Project Naming & Branding

- **Product name:** BizzAR
- **Domain (planned):** bizzar.app
- **AR viewer URL format:** `bizzar.app/ar/[userId]/[cardId]`
- **Watermark text:** "Made with BizzAR"
