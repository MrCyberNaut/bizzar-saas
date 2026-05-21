# BizzAR — Implementation Plan
**Date:** 2026-04-06  
**Spec:** 2026-04-06-bizzar-design.md  
**Stack:** Next.js 14 · TypeScript · Tailwind · R3F · 8th Wall OSS · A-Frame · Supabase · Stripe

---

## Phases Overview

| Phase | Name | Depends On |
|---|---|---|
| 1 | Project Scaffold & Config | — |
| 2 | Supabase Schema & RLS | 1 |
| 3 | Auth & Dashboard Shell | 2 |
| 4 | Card Creation & Image Upload | 3 |
| 5 | Image Target Compilation (Edge Function) | 4 |
| 6 | 3D Hub Editor (R3F) | 4 |
| 7 | 2D Panel Grid Editor | 6 |
| 8 | Module System — Config & Rendering | 7 |
| 9 | AR Viewer (Standalone HTML + 8th Wall + A-Frame) | 5, 8 |
| 10 | Freemium Gate & Stripe | 3, 9 |
| 11 | Publishing, QR Code & Sharing | 9, 10 |
| 12 | Landing Page & Pricing | 10 |

---

## Phase 1 — Project Scaffold & Config

**Goal:** Runnable Next.js 14 app with all dependencies installed, folder structure established, Supabase client wired up.

### Tasks

**1.1 — Init Next.js project**
```
npx create-next-app@14 bizzar --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

**1.2 — Install core dependencies**
```
npm install @supabase/supabase-js @supabase/ssr
npm install @react-three/fiber @react-three/drei three
npm install @types/three
npm install framer-motion
npm install stripe @stripe/stripe-js
npm install qrcode qrcode.react
npm install @types/qrcode
npm install zustand
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

**1.3 — Folder structure**
```
src/
  app/
    (marketing)/          # landing, pricing — no auth layout
      page.tsx
      pricing/page.tsx
      layout.tsx
    (app)/                # auth-gated — dashboard, editor
      dashboard/
        page.tsx
        cards/page.tsx
      editor/
        [cardId]/page.tsx
      layout.tsx
    ar/
      [userId]/
        [cardId]/
          page.tsx        # serves standalone HTML — see Phase 9
  components/
    ui/                   # reusable primitives (Button, Input, Badge, etc.)
    editor/               # 3D hub + 2D panel editor components
    modules/              # per-module render components (editor + AR)
    ar/                   # AR viewer components
  lib/
    supabase/
      client.ts           # browser client
      server.ts           # server client (RSC / route handlers)
      middleware.ts
    stripe/
      client.ts
      webhook.ts
    8thwall/
      compile.ts          # image target compilation helpers
  types/
    database.ts           # generated Supabase types
    modules.ts            # module type definitions + config shapes
  store/
    editor.ts             # Zustand store for editor state
```

**1.4 — Environment variables**
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

**1.5 — Supabase middleware (session refresh)**
Set up `middleware.ts` at project root using `@supabase/ssr` to handle session cookie refresh on every request. Protect `(app)` routes — redirect to `/` if not authenticated.

**1.6 — TypeScript path aliases & Tailwind config**
Confirm `@/*` alias works. Set up Tailwind with custom design tokens if needed.

**Verification:** `npm run dev` runs with no errors. `/` loads. Route groups resolve correctly.

---

## Phase 2 — Supabase Schema & RLS

**Goal:** All tables created, types generated, RLS locked down.

### Tasks

**2.1 — Create tables via Supabase migration**

```sql
-- users (extends auth.users)
create table public.users (
  id uuid primary key references auth.users(id) on delete cascade,
  email text not null,
  plan text not null default 'free' check (plan in ('free', 'pro')),
  stripe_customer_id text,
  created_at timestamptz default now()
);

-- cards
create table public.cards (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.users(id) on delete cascade,
  name text not null,
  card_image_url text,
  image_target_url text,
  published boolean not null default false,
  created_at timestamptz default now()
);

-- panels
create table public.panels (
  id uuid primary key default gen_random_uuid(),
  card_id uuid not null references public.cards(id) on delete cascade,
  slot text not null check (slot in ('overlay', 'top', 'left', 'right')),
  enabled boolean not null default false,
  grid_cols int2 not null default 4,
  grid_rows int2 not null default 4
);

-- modules
create table public.modules (
  id uuid primary key default gen_random_uuid(),
  panel_id uuid not null references public.panels(id) on delete cascade,
  type text not null check (type in ('vcard','socials','github','image_gallery','video','animated_overlay','calendly','bio')),
  grid_x int2 not null,
  grid_y int2 not null,
  grid_w int2 not null,
  grid_h int2 not null,
  config jsonb not null default '{}'
);
```

**2.2 — Triggers**

```sql
-- Auto-create user row on auth signup
create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public.users (id, email)
  values (new.id, new.email);
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

-- Auto-create 4 panels on card creation
create or replace function public.handle_new_card()
returns trigger as $$
begin
  insert into public.panels (card_id, slot, enabled) values
    (new.id, 'overlay', true),
    (new.id, 'top', true),
    (new.id, 'left', false),
    (new.id, 'right', false);
  return new;
end;
$$ language plpgsql security definer;

create trigger on_card_created
  after insert on public.cards
  for each row execute procedure public.handle_new_card();
```

**2.3 — RLS policies**

```sql
-- users: own row only
alter table public.users enable row level security;
create policy "users: own row" on public.users
  for all using (auth.uid() = id);

-- cards: own cards + public read for published
alter table public.cards enable row level security;
create policy "cards: own" on public.cards
  for all using (auth.uid() = user_id);
create policy "cards: public read if published" on public.cards
  for select using (published = true);

-- panels: via card ownership + public read if card published
alter table public.panels enable row level security;
create policy "panels: own via card" on public.panels
  for all using (
    exists (select 1 from public.cards where id = panels.card_id and user_id = auth.uid())
  );
create policy "panels: public read if card published" on public.panels
  for select using (
    exists (select 1 from public.cards where id = panels.card_id and published = true)
  );

-- modules: via panel → card ownership + public read if published
alter table public.modules enable row level security;
create policy "modules: own via panel" on public.modules
  for all using (
    exists (
      select 1 from public.panels p
      join public.cards c on c.id = p.card_id
      where p.id = modules.panel_id and c.user_id = auth.uid()
    )
  );
create policy "modules: public read if card published" on public.modules
  for select using (
    exists (
      select 1 from public.panels p
      join public.cards c on c.id = p.card_id
      where p.id = modules.panel_id and c.published = true
    )
  );
```

**2.4 — Storage buckets**
- `card-images` — public read, auth write (card image uploads)
- `image-targets` — public read, service role write only (compiled .mind files)
- `module-media` — public read, auth write (images/videos for modules)

**2.5 — Generate TypeScript types**
```
npx supabase gen types typescript --project-id <id> > src/types/database.ts
```

**Verification:** All tables visible in Supabase dashboard. RLS policies listed. Types file generated with correct shape.

---

## Phase 3 — Auth & Dashboard Shell

**Goal:** Users can sign up, log in, see an empty dashboard.

### Tasks

**3.1 — Auth pages**
- `/` with "Sign in" CTA → Supabase magic link + Google OAuth
- Auth callback handler at `/auth/callback`
- Redirect to `/dashboard` on success

**3.2 — Dashboard layout**
- Sidebar: logo, nav links (Cards, Settings, Upgrade), user avatar + plan badge
- Main content area

**3.3 — Dashboard cards page (`/dashboard`)**
- Fetch `cards` for current user (ordered by `created_at` desc)
- Card grid: each card shows name, published status badge, "Edit" button → `/editor/[cardId]`, "View AR" link
- Empty state: illustration + "Create your first card" CTA
- "New Card" button → triggers card creation modal (Phase 4)

**3.4 — Zustand editor store scaffold**
```typescript
// src/store/editor.ts
interface EditorStore {
  card: Card | null
  panels: Panel[]
  modules: Module[]
  selectedPanelId: string | null
  setCard: (card: Card) => void
  setPanels: (panels: Panel[]) => void
  setModules: (modules: Module[]) => void
  selectPanel: (id: string | null) => void
  upsertModule: (module: Module) => void
  removeModule: (id: string) => void
}
```

**Verification:** Sign up flow works. Dashboard loads. Empty state shown. User row created in `users` table.

---

## Phase 4 — Card Creation & Image Upload

**Goal:** User can create a card, upload an image, and see it in the dashboard.

### Tasks

**4.1 — New card modal**
- Name input
- Card image upload (drag-drop or click)
- Preview of uploaded image
- Submit → creates `cards` row → 4 `panels` auto-created by trigger → redirect to `/editor/[cardId]`

**4.2 — Image upload to Supabase Storage**
- Upload to `card-images/[userId]/[cardId].jpg`
- Update `cards.card_image_url` after upload
- Show upload progress

**4.3 — Card image upload validation**
- Accepted: jpg, png, webp
- Max size: 5MB
- Minimum resolution: 400×200px (too small = bad tracking)
- Warn if image is very low contrast (check pixel variance — simple heuristic)

**4.4 — Types for modules**
```typescript
// src/types/modules.ts
export type ModuleType = 'vcard' | 'socials' | 'github' | 'image_gallery' | 'video' | 'animated_overlay' | 'calendly' | 'bio'

export type ModuleSizeVariant = { w: number; h: number; label: string }

export const MODULE_SIZE_VARIANTS: Record<ModuleType, ModuleSizeVariant[]> = {
  vcard:            [{ w:1, h:1, label:'1×1' }, { w:2, h:1, label:'2×1' }, { w:4, h:1, label:'4×1' }],
  socials:          [{ w:1, h:1, label:'1×1' }, { w:2, h:1, label:'2×1' }, { w:4, h:1, label:'4×1' }],
  bio:              [{ w:1, h:1, label:'1×1' }, { w:2, h:1, label:'2×1' }, { w:2, h:2, label:'2×2' }, { w:4, h:1, label:'4×1' }, { w:4, h:2, label:'4×2' }],
  github:           [{ w:2, h:1, label:'2×1' }, { w:4, h:1, label:'4×1' }, { w:4, h:2, label:'4×2' }],
  image_gallery:    [{ w:1, h:1, label:'1×1' }, { w:2, h:1, label:'2×1' }, { w:2, h:2, label:'2×2' }, { w:4, h:2, label:'4×2' }, { w:4, h:4, label:'4×4' }],
  video:            [{ w:1, h:1, label:'1×1' }, { w:2, h:2, label:'2×2' }, { w:2, h:3, label:'2×3' }, { w:4, h:2, label:'4×2' }, { w:4, h:4, label:'4×4' }],
  animated_overlay: [{ w:4, h:4, label:'Full panel' }],
  calendly:         [{ w:1, h:1, label:'1×1' }, { w:2, h:1, label:'2×1' }, { w:4, h:1, label:'4×1' }, { w:4, h:2, label:'4×2' }],
}
```

**Verification:** Card created in DB. Image visible in Supabase Storage. 4 panels created by trigger. Redirect to editor works.

---

## Phase 5 — Image Target Compilation (Edge Function)

**Goal:** After card image upload, a `.mind` file is automatically compiled and stored.

### Tasks

**5.1 — Supabase Edge Function: `compile-image-target`**
- Triggered by: HTTP POST from app after card image upload (not a Storage trigger — Deno runtime limitation with CLI tools)
- Input: `{ cardId, imageUrl }`
- Steps:
  1. Download image from Supabase Storage
  2. Run 8th Wall `image-target-cli` to compile `.mind` file
  3. Upload `.mind` to `image-targets/[cardId].mind`
  4. Update `cards.image_target_url`
- Note: `image-target-cli` is a Node.js tool. Verify Deno compatibility or use a lightweight Docker sidecar / Next.js API route with Node runtime instead.

**5.2 — Fallback: Next.js API Route (if Deno incompatible)**
- `POST /api/compile-image-target`
- Uses `edge: false` (Node.js runtime)
- Same logic as above

**5.3 — Processing state in UI**
- After upload, card shows "Processing..." state
- Poll `cards.image_target_url` every 2s until populated
- Then enable "Open Editor" button
- Error state if compilation fails

**Verification:** Upload card image → `.mind` file appears in Supabase Storage `image-targets/` bucket → `cards.image_target_url` populated.

---

## Phase 6 — 3D Hub Editor (R3F)

**Goal:** Isometric 3D view of card + 4 floating panels. Click panel to select it.

### Tasks

**6.1 — Editor page layout (`/editor/[cardId]`)**
- Server component fetches card + panels + modules, hydrates Zustand store
- Split layout: R3F canvas (left/main) + panel editor drawer (right, hidden until panel selected)
- Header: card name, Publish button, Back to dashboard

**6.2 — R3F scene setup**
- `<Canvas>` with orthographic camera at isometric angle (~45° elevation, ~45° azimuth)
- `<OrbitControls>` (Drei) — allow rotate + zoom, disable pan
- Ambient + directional lighting
- Background: dark neutral

**6.3 — Business card mesh**
- `<mesh>` with `PlaneGeometry` sized to standard business card ratio (3.5:2 inches → 3.5:2 units)
- `<meshStandardMaterial>` with card image as texture (loaded from `card_image_url`)
- Slight elevation shadow

**6.4 — Panel planes (4 slots)**

Each panel is a `<mesh>` with a semi-transparent material. Positions are hardcoded constants:

```typescript
export const PANEL_TRANSFORMS = {
  overlay: { position: [0, 0.01, 0],    rotation: [-Math.PI/2, 0, 0], scale: [3.5, 2, 1] },
  top:     { position: [0, 1.2, -0.8],  rotation: [-Math.PI/4, 0, 0], scale: [3.5, 2, 1] },
  left:    { position: [-2.2, 1.2, 0],  rotation: [-Math.PI/4, Math.PI/6, 0], scale: [2, 2, 1] },
  right:   { position: [2.2, 1.2, 0],   rotation: [-Math.PI/4, -Math.PI/6, 0], scale: [2, 2, 1] },
}
```
(Fine-tune visually — these are approximate)

**6.5 — Panel interaction**
- Click unlocked panel → `selectPanel(id)` in Zustand → drawer opens
- Hover: panel brightens slightly
- Locked panels (free plan, left/right): dimmed material + lock icon via `<Html>` (Drei)
- Click locked panel → upgrade modal

**6.6 — Panel content preview in 3D**
- Once modules are placed, show simplified preview on panel plane using `<Html>` (Drei) — scaled to fit panel size
- This is a visual preview only, not interactive

**Verification:** 3D scene renders. Card texture loads. Panels visible at correct positions. Clicking panel selects it. Locked panels show Pro badge.

---

## Phase 7 — 2D Panel Grid Editor

**Goal:** Click a panel → drawer opens with 4×4 grid. Drag modules from tray onto grid. Autosave.

### Tasks

**7.1 — Panel editor drawer**
- Slides in from right (Framer Motion)
- Header: panel slot name, close button
- Two-column layout: module tray (left 1/3) + grid (right 2/3)

**7.2 — 4×4 grid**
- CSS grid, 4 columns × 4 rows, fixed cell size
- Each cell renders as an empty slot (dashed border)
- Occupied cells render the module tile spanning `grid_w × grid_h`

**7.3 — Module tray**
- List of all module types
- Free modules: normal
- Pro modules (free plan): shown with lock icon + "Pro" badge, not draggable
- Each tray item is draggable via `@dnd-kit/core`

**7.4 — Drag and drop**
- `DndContext` wraps tray + grid
- On `DragEnd`:
  1. Find target cell from drop coordinates
  2. Check if user's plan allows this module type
  3. Check all cells in `[x..x+w, y..y+h]` are empty (collision detection)
  4. If valid: show size picker modal → user selects variant → module placed
  5. If collision: snap back (no state change)

**7.5 — Size picker modal**
- Triggered after successful drop (before committing)
- Shows available size variants for that module type as selectable pills
- Confirm → place at selected size

**7.6 — Module config drawer**
- Click a placed module → config form slides in
- Forms per module type (see Phase 8)
- Save → upsert `modules` row in Supabase + update Zustand

**7.7 — Autosave**
- Every module placement, removal, and config change triggers an immediate Supabase upsert
- Optimistic UI — update Zustand first, then sync
- Show subtle "Saved" toast on success

**7.8 — Remove module**
- Right-click or trash icon on placed module → remove
- Delete from Supabase + update Zustand

**Verification:** Drag module onto grid, size picker appears, module placed. Config form opens. Supabase `modules` table updated. Collision detection works. Free user cannot place Pro modules.

---

## Phase 8 — Module System (Config Forms + Rendering)

**Goal:** Each module type has a config form (editor) and a render component (AR viewer).

### Tasks

**8.1 — Module config forms (one per type)**

Each form reads/writes the module's `config` JSONB. Autosaves on change.

- `VCardForm` — name, email, phone, company, website inputs
- `SocialsForm` — list of platform + URL pairs, add/remove
- `BioForm` — textarea (max 280 chars)
- `GitHubForm` — username input
- `ImageGalleryForm` — up to 4 image uploads (to `module-media/` Storage), alt text
- `VideoForm` — file upload (mp4, max 20MB) or YouTube/Vimeo URL input
- `AnimatedOverlayForm` — mp4 or GIF upload (max 20MB)
- `CalendlyForm` — URL input

**8.2 — Module preview components (in grid)**

Small tile renders inside the grid cell — shows module type icon + label. Not full renders, just identifiers.

**8.3 — Module AR render components**

Full render components used by the AR viewer. These are plain HTML (no React) — output raw HTML strings or DOM elements:

- `renderVCard(config)` → button that triggers `.vcf` download
- `renderSocials(config)` → row of icon links
- `renderBio(config)` → styled text block
- `renderGitHub(config)` → fetches GitHub contributions API, renders contribution graph as SVG
- `renderImageGallery(config)` → 2D image grid, tap to expand
- `renderVideo(config)` → `<video autoplay loop muted playsinline>` or embedded iframe
- `renderAnimatedOverlay(config)` → `<video>` or `<img>` fullscreen on panel plane
- `renderCalendly(config)` → tap button that opens Calendly URL

**Verification:** Each config form saves correctly to Supabase. Module previews show in grid. AR render functions output valid HTML.

---

## Phase 9 — AR Viewer (Standalone HTML + 8th Wall + A-Frame)

**Goal:** `/ar/[userId]/[cardId]` loads in mobile browser, detects business card, renders panels.

### Tasks

**9.1 — Next.js route that serves raw HTML**

`/app/ar/[userId]/[cardId]/page.tsx` — a server component that returns a full `<html>` document (not wrapped in root layout). Use `export const metadata` carefully — this page breaks out of the normal layout.

Alternative: use a Next.js API route (`route.ts`) that returns `text/html` response. This is cleaner for a completely standalone page.

**9.2 — AR page structure**

```html
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="https://cdn.jsdelivr.net/npm/aframe@1.4.0/dist/aframe.min.js"></script>
  <script src="<!-- 8th Wall Image Targets CDN or self-hosted -->"></script>
</head>
<body style="margin:0;overflow:hidden;">
  <a-scene
    xrextras-loading
    xrextras-runtime-error
    xrweb="disableWorldTracking: true">
    <a-assets><!-- preload card image --></a-assets>
    <xrextras-named-image-target name="business-card">
      <!-- panels injected here by JS -->
    </xrextras-named-image-target>
  </a-scene>
  <script>
    // fetch card config from Supabase REST
    // register .mind file
    // inject panel entities
    // render modules as HTML overlays
  </script>
</body>
</html>
```

**9.3 — Fetch card config (no SDK, plain fetch)**

```javascript
const { data } = await fetch(
  `${SUPABASE_URL}/rest/v1/cards?id=eq.${cardId}&published=eq.true&select=*,panels(*,modules(*))`,
  { headers: { apikey: SUPABASE_ANON_KEY } }
).then(r => r.json())
```

**9.4 — Register image target**

```javascript
// Register the .mind file dynamically
XRExtras.NamedImageTarget.configure({
  imageTargets: [{ name: 'business-card', mindFile: card.image_target_url }]
})
```

**9.5 — Inject panel A-Frame entities**

For each enabled panel, create `<a-entity>` with position + rotation from `PANEL_TRANSFORMS` constants (same values as editor). Append to the `<xrextras-named-image-target>`.

**9.6 — Render modules as HTML overlays**

Use A-Frame's `<a-entity>` with `xrextras-attach` component or position HTML overlays using A-Frame's `<a-plane>` with `material="shader: html"` approach. Each module's render function (Phase 8.3) outputs HTML injected into the overlay.

**9.7 — Watermark**

For free plan cards: fixed-position HTML badge overlay "Made with BizzAR" in bottom corner. Read from card config — `card.user.plan` included in the Supabase query.

**9.8 — Error states**
- Card not found / not published → friendly error screen
- Camera permission denied → instruction screen
- 8th Wall load failure → fallback message

**Verification:** Open `/ar/[userId]/[cardId]` on mobile. Camera opens. Point at business card. Panels appear anchored to card. Module interactions work. Watermark shows for free cards.

---

## Phase 10 — Freemium Gate & Stripe

**Goal:** Free/pro plans enforced. Stripe checkout works. Plan upgrade unlocks panels in real time.

### Tasks

**10.1 — Stripe products setup**
- Create "BizzAR Pro" subscription product in Stripe dashboard
- Monthly price ID → store in env var `STRIPE_PRO_PRICE_ID`

**10.2 — Stripe Checkout flow**
- "Upgrade to Pro" button → POST `/api/stripe/checkout`
- Creates Stripe Checkout session (subscription mode)
- Redirect to Stripe hosted checkout
- Success redirect → `/dashboard?upgraded=true`

**10.3 — Stripe webhook handler**
- `POST /api/stripe/webhook`
- Verify signature with `stripe.webhooks.constructEvent`
- Handle `checkout.session.completed`:
  1. Get customer email / metadata
  2. Look up user by email
  3. Set `users.plan = 'pro'`
  4. Set `panels.enabled = true` for `left` and `right` on all their cards
- Handle `customer.subscription.deleted`:
  1. Set `users.plan = 'free'`
  2. Set `panels.enabled = false` for `left` and `right` on all their cards

**10.4 — Realtime plan upgrade in editor**
- In editor, subscribe to Supabase Realtime on `users` row
- On `plan` change → re-fetch panels → unlock left/right panels in Zustand → panel planes update in R3F scene

**10.5 — UI gate enforcement**
- Module tray: Pro modules show lock icon for free users, not draggable
- Panel planes: left/right dimmed with Pro badge for free users
- "New Card" button: disabled with tooltip for free users who have 1 card already
- All gates read from `users.plan` fetched server-side — not client state

**Verification:** Free user cannot drag pro modules. Stripe checkout completes. Webhook fires. `users.plan` updated to 'pro'. Editor unlocks panels without page reload.

---

## Phase 11 — Publishing, QR Code & Sharing

**Goal:** Publish button makes AR live. QR code generated. Shareable URL shown.

### Tasks

**11.1 — Publish / Unpublish**
- Publish button in editor header: sets `cards.published = true` via Supabase
- Disabled if `image_target_url` is null (compilation not done yet)
- On publish: show success state with shareable URL + QR

**11.2 — Shareable URL**
```
bizzar.app/ar/[userId]/[cardId]
```
- Copy to clipboard button
- Open in new tab button

**11.3 — QR Code**
- Generated client-side with `qrcode.react`
- Points to AR viewer URL
- Download as PNG button
- Show in a modal after publish with print instructions

**11.4 — Dashboard published state**
- Published cards show green "Live" badge + "View AR" link
- Unpublished cards show grey "Draft" badge
- "Unpublish" option in card menu (sets `published = false`)

**Verification:** Publish sets DB flag. AR URL accessible publicly. QR code scans to correct URL. Unpublish makes AR route 404.

---

## Phase 12 — Landing Page & Pricing

**Goal:** Marketing pages explaining BizzAR, pricing tiers, conversion to signup.

### Tasks

**12.1 — Landing page (`/`)**
- Hero: headline, subheadline, CTA ("Create your free card")
- How it works: 3-step visual (upload → build → share)
- Feature highlights: module showcase
- Testimonials placeholder
- Footer

**12.2 — Pricing page (`/pricing`)**
- Free vs Pro comparison table (matches freemium gate spec)
- "Get started free" + "Upgrade to Pro" CTAs
- FAQ section

**12.3 — Navigation**
- Top nav: Logo, Pricing, Sign in / Dashboard

**Verification:** Landing page loads. Pricing table correct. CTAs route correctly.

---

## Implementation Order Recommendation

Start with Phases 1 → 2 → 3 in a single session to get the scaffold and auth solid. Then Phase 4 + 5 together (card creation + image target compilation). Then 6 + 7 (3D hub + grid editor). Then 8 (module system). Then 9 (AR viewer — the most independent piece). Then 10 + 11 (payments + publishing). Then 12 last (marketing, can be parallel with testing).

**The AR viewer (Phase 9) is the highest-risk phase** — 8th Wall OSS integration on mobile is the most unknown. Build a proof-of-concept spike (just card detection, no modules) before committing to the full module rendering approach.

---

## Definition of Done

- [ ] User can sign up, upload card, build panels with modules, publish
- [ ] Recipient can scan QR → browser opens → point at card → panels appear in AR
- [ ] All 8 module types configurable and rendering in AR
- [ ] Free plan limited to 2 panels + 3 modules + 1 card + watermark
- [ ] Pro plan unlocks in real time via Stripe webhook
- [ ] RLS prevents any cross-user data access
- [ ] AR viewer works on iOS Safari and Android Chrome
- [ ] Card image quality warning shown on upload
