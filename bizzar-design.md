# BizzAR — Product Design Spec
**Date:** 2026-04-06  
**Status:** Approved  
**Tagline:** Business cards with AR — point, scan, connect.

---

## 1. Product Overview

BizzAR is a WebAR SaaS that turns physical business cards into interactive augmented reality experiences. A professional uploads their business card, builds a set of floating panels around it using a drag-and-drop module editor, and publishes a shareable AR experience. Anyone who receives the physical card points their phone camera at it — no app install required — and sees the panels come to life anchored to the card in 3D space.

**Core target user:** Individual professionals — freelancers, consultants, founders.  
**AR trigger method:** Image tracking via 8th Wall OSS Image Targets. Recipient points phone camera at physical card; AR launches via QR code printed on the card opening a mobile browser URL.  
**Delivery:** Pure WebAR — opens in mobile browser, zero app install friction.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Frontend / App Router | Next.js 14 (TypeScript, Tailwind CSS) |
| 3D Hub Editor | React Three Fiber (R3F) + Drei |
| AR Runtime | 8th Wall OSS Image Targets + A-Frame |
| Animation / Motion | Framer Motion (UI), R3F for 3D |
| Backend | Supabase (Postgres, Auth, Storage, Edge Functions) |
| Payments | Stripe (subscription, webhook) |
| QR Generation | qrcode.js (client-side, no server needed) |
| Image Target Compilation | 8th Wall image-target-cli (Edge Function) |

**Key architectural decision:** The AR viewer (`/ar/[userId]/[cardId]`) is a fully standalone HTML page — no React, no Next.js shell. 8th Wall + A-Frame are loaded via script tags. This keeps the AR bundle tiny for fast mobile load and avoids React DOM conflicts with AR runtimes.

---

## 3. Routes & Pages

| Route | Description |
|---|---|
| `/` | Marketing landing page |
| `/pricing` | Pricing / plan comparison |
| `/dashboard` | Card list, create new card |
| `/dashboard/cards` | Card management |
| `/editor/[cardId]` | 3D Hub editor + 2D panel editor |
| `/ar/[userId]/[cardId]` | Standalone AR viewer (no Next.js shell) |

Publishing is **path-based, not subdomain-based** at MVP. This eliminates hosting cost complexity. Vanity subdomains are a future Pro feature.

---

## 4. Data Model

### `users`
Extended from Supabase Auth (`auth.users`).

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | auth.users FK, RLS anchor |
| email | text | |
| plan | enum | `free` \| `pro` — source of truth for gating |
| stripe_customer_id | text | nullable, set on first checkout |
| created_at | timestamptz | |

### `cards`
One business card per row — the root entity.

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| user_id | uuid FK | → users.id |
| name | text | User-given label e.g. "My Freelance Card" |
| card_image_url | text | Supabase Storage path — original upload |
| image_target_url | text | Compiled .mind file from 8th Wall CLI |
| published | boolean | Controls whether /ar/... route is live |
| created_at | timestamptz | |

### `panels`
4 panels created automatically when a card is created.

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| card_id | uuid FK | → cards.id |
| slot | enum | `overlay` \| `top` \| `left` \| `right` |
| enabled | boolean | Free: only `overlay` + `top` enabled |
| grid_cols | int2 | Panel grid width in cells (default: 4) |
| grid_rows | int2 | Panel grid height in cells (default: 4) |

### `modules`
Each dragged module placed on a panel grid.

| Field | Type | Notes |
|---|---|---|
| id | uuid PK | |
| panel_id | uuid FK | → panels.id |
| type | enum | `vcard` \| `socials` \| `github` \| `image_gallery` \| `video` \| `animated_overlay` \| `calendly` \| `bio` |
| grid_x | int2 | Top-left cell column |
| grid_y | int2 | Top-left cell row |
| grid_w | int2 | Width in cells |
| grid_h | int2 | Height in cells |
| config | jsonb | Module-specific data (links, media URLs, usernames etc.) |

### Relationships
- `users` → has many → `cards`
- `cards` → has exactly 4 → `panels` (auto-created on card init)
- `panels` → has many → `modules` (drag-dropped by user)
- `users.plan` → gates → `panels.enabled` + available module types

### RLS Policy Summary
- Users can only read/write their own rows across all tables.
- AR viewer route fetches card config via a public-safe read (published cards only, no auth required).
- Freemium gate is enforced server-side via RLS + `users.plan` — never client-side.

---

## 5. 3D Panel Layout

The 3D Hub editor shows the business card in an isometric/top-slant view (like Blender). Four panels float around it at fixed spatial positions:

| Slot | Position | Description |
|---|---|---|
| `overlay` | Directly on top of card | Overlays the card surface. For animated video overlay module. |
| `top` | Above card, 45° tilted | Nintendo DS-style tilted panel facing viewer |
| `left` | Left of top panel | Wing panel, angled to face viewer |
| `right` | Right of top panel | Wing panel, angled to face viewer |

**Free plan:** `overlay` + `top` only. `left` and `right` shown as dimmed planes with a Pro lock badge.  
**Pro plan:** All 4 panels unlocked.

In the AR viewer (A-Frame), panels are positioned as `<a-entity>` children of the image target anchor, with transforms pre-calculated to match the editor layout exactly.

---

## 6. Panel Grid & Module System

### Grid
Each panel is a **4-column × 4-row magnetic grid**. Modules snap to cells on drop, identical to the iOS home screen widget system.

### Module Size Variants
Modules are not freely resizable. Each module type has a set of **preset size variants** (named by `grid_w × grid_h`). The user picks a variant when placing. The DB stores raw `grid_w` / `grid_h` — size variant names are UI-only.

| Module | Plan | Min Size | Available Sizes |
|---|---|---|---|
| Add to Contacts (vCard) | Free | 1×1 | 1×1, 2×1, 4×1 |
| Social Icons | Free | 1×1 | 1×1, 2×1, 4×1 |
| Bio / Text Block | Free | 1×1 | 1×1, 2×1, 2×2, 4×1, 4×2 |
| GitHub Graph | Pro | 2×1 | 2×1, 4×1, 4×2 |
| Image Gallery | Pro | 1×1 | 1×1, 2×1, 2×2, 4×2, 4×4 |
| Video Panel | Pro | 1×1 | 1×1, 2×2, 2×3, 4×2, 4×4 |
| Animated Overlay | Pro | fills panel | fills overlay panel only |
| Calendly | Pro | 1×1 | 1×1, 2×1, 4×1, 4×2 |

### Grid Rules
- **Snap on drop:** Module snaps to nearest valid grid position.
- **Collision detection:** No overlapping. Each cell is occupied by at most one module.
- **Bounce back:** If target cells are occupied, module returns to tray (iOS behaviour).
- **Resize in place:** User taps a placed module → switches size variant. If new size doesn't fit, prompt to move or cancel.
- **Autosave:** Every grid change syncs to Supabase automatically.

### Module `config` JSONB shapes (per type)
```
vcard:     { name, email, phone, company, website }
socials:   { links: [{ platform, url }] }
bio:       { text }
github:    { username }
image_gallery: { images: [{ url, alt }] }
video:     { url, type: 'upload' | 'youtube' | 'vimeo' }
animated_overlay: { url, type: 'mp4' | 'gif' }
calendly:  { url }
```

---

## 7. Freemium Gate

| Feature | Free | Pro |
|---|---|---|
| Panels | overlay + top only | All 4 panels |
| Free modules (vCard, Socials, Bio) | ✓ | ✓ |
| Pro modules (GitHub, Image, Video, Overlay, Calendly) | Locked / greyed out | ✓ |
| Cards | 1 card | Unlimited |
| Watermark in AR view | Shown | Removed |
| Vanity URL | No | Coming later |

**Gate enforcement:** `users.plan` is read server-side via Supabase RLS. The editor UI reflects locked state visually (dimmed panels, disabled module tiles) but the actual enforcement is at the data layer — locked panels have `enabled = false` and are excluded from the AR viewer query.

**Watermark:** A small "Made with BizzAR" badge overlaid in the A-Frame AR scene for free users. Removed when `plan = pro`. Acts as an organic growth loop — every AR experience shared is a referral.

---

## 8. Creator Flow

1. **Sign up** — Magic link or Google OAuth via Supabase Auth. `plan = free` by default.
2. **Upload card** — Upload front-of-card image. Supabase Storage receives it. Edge Function fires: runs 8th Wall `image-target-cli` → compiles `.mind` file → stores `image_target_url` on the card row.
3. **Enter Hub editor** — Card rendered in isometric R3F scene. 4 panel planes floating in correct spatial positions. Free panels active, pro panels dimmed.
4. **Build panels** — Click panel → 2D grid editor opens as a drawer. Drag module from tray → pick size → configure via form. Autosave on every change.
5. **Publish** — Hit Publish → `cards.published = true`. AR URL goes live. Dashboard shows shareable URL + generated QR code.
6. **Upgrade (optional)** — Stripe Checkout. On `checkout.session.completed` webhook → `users.plan = pro`. Supabase Realtime pushes plan change to editor UI — locked panels unlock instantly, no reload.

---

## 9. Recipient Flow (AR Experience)

1. Recipient receives physical business card with QR printed on it.
2. Scans QR → opens `/ar/[userId]/[cardId]` in mobile browser.
3. Page loads: fetches card config + `.mind` file from Supabase. Boots 8th Wall Image Targets + A-Frame. Requests camera permission.
4. 8th Wall registers `.mind` file as image target. Scans camera feed.
5. Card detected → A-Frame spawns panel `<a-entity>` elements at correct 3D transforms relative to card anchor.
6. Modules render — HTML overlays for interactive elements (vCard download, social taps), A-Frame planes for media.
7. Recipient taps modules to interact: download contact, open social link, view video, open Calendly.

---

## 10. Publishing & Hosting

- **AR URL format:** `bizzar.app/ar/[userId]/[cardId]`
- **Path-based routing** at MVP — no subdomains, no extra hosting cost.
- **QR Code:** Generated client-side via `qrcode.js`. User downloads PNG to print.
- **Unpublish:** Sets `published = false` → AR route returns 404.
- **Re-edit after publish:** Changes saved to DB immediately. User must re-publish to push changes live (or autosave can update live — TBD based on UX preference).
- **Future:** Vanity subdomain (`username.bizzar.app`) as a Pro feature.

---

## 11. Out of Scope for MVP

- Native app wrapper
- SLAM / surface tracking (Image Targets only)
- Multi-card sharing / team accounts
- Analytics (scan counts, module tap rates)
- Custom domain / subdomain
- A/B panel variants
- Offline mode
- Backend module rendering (all modules render client-side in AR)

---

## 12. Open Questions

- **Re-publish UX:** Should edits go live instantly (autosave → live) or require an explicit re-publish step? Explicit publish is safer and gives the user control.
- **Image target quality threshold:** 8th Wall CLI gives a quality score for image targets. Should we warn users if their card image is too low-contrast for reliable tracking?
- **Video upload size limit:** Set a hard cap (e.g. 20MB) on video uploads to Supabase Storage to manage costs on the free tier.
