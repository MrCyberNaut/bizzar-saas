# BizzAR — Project Instructions for Claude Code & Agents

> 🚨 **PRIMARY AGENT DIRECTIVE:** Read `AGENT_MASTER.md` before doing ANY work. It is the synchronized single source of truth for repository structure, branches, AR engine math, WebGL shaders, tracking pipelines, and active state.

## Model Tier Rules (MANDATORY)

Always use the right model for the task — applies to all subagents spawned:

| Task | Model |
|---|---|
| Thinking, architecture, decisions, complex reasoning | **Opus** |
| Coding, implementation, debugging, file edits | **Sonnet** |
| Writing logs, memory writes, file appends, rote output | **Haiku** |

**Error Log:** Maintained at `d:/Projects/BizzAR-SaaS/ERROR_LOG.md`. Every error encountered must be logged. When fixed, update the entry with the fix.

---

## Strategy & Current Focus

**Current Goal:** Get the AR engine on physical business cards working with absolute excellence. The SaaS layer is parked.
- **Active Engine Repo:** `bizzar-proto` (Next.js 14, React Three Fiber, MindAR, custom WebGL preprocessing shaders, FFT phase correlation fallback, glassmorphic 3D panels).
- **Parked SaaS Repo:** `bizzar` (Next.js 14, Supabase, Stripe, DnD 3D Hub Editor).
- **Master Agent Reference:** Always check and maintain `AGENT_MASTER.md`.
- **User Explainer Files:** Maintain `PROJECT_EXPLAINER.txt` and `PROJECT_EXPLAINER.html` at session milestones.

---

## Project Layout

```
d:/Projects/BizzAR-SaaS/
  AGENT_MASTER.md   ← Master technical truth across Antigravity & Claude Code
  PROJECT_EXPLAINER.txt ← Plain-text project status for Arnav
  PROJECT_EXPLAINER.html ← Interactive UI dashboard for Arnav
  bizzar-proto/     ← AR engine prototype (R3F + MindAR, WebGL shaders, Phase Correlation) [ACTIVE FOCUS]
  bizzar/           ← Main SaaS app (Next.js 14, Supabase, Stripe, 3D Editor) [PARKED]
  knowledge/        ← 30 Computer Vision & WebAR research papers (00-29 + PDFs)
  bizzar-design.md  ← Full product design spec
  bizzar-implementation-plan.md
```

---

## Current State

- Auth bypassed for testing (TODO markers in code — re-enable before prod)
- RLS disabled on Supabase tables — re-enable before prod
- Supabase project: `wyujyqrblovrmnymfmms` (region: ap-southeast-2) — prone to pausing on free tier
- Dev server runs on port 3001 (port 3000 used by FAPRA project)

---

## Session Start Checklist

At the start of every session on this project:
1. Query `graphify-out/graph.json` or `graphify-out/GRAPH_REPORT.md` — structure first
2. Read `bizzar-design.md` for product context
3. Check Supabase project status — may need restoring if inactive
4. Check `bizzar-proto/` for any engine code reusable for current task

## Knowledge Graph (graphify-out/)

Graph built 2026-05-20: 309 nodes · 435 edges · 31 communities.

**God nodes (most connected core abstractions):**
- `ModuleConfig` (17 edges) — central type for all AR module data
- `EditorClient()` (16 edges) — main 3D editor component
- `Database` (10 edges) — Supabase client
- `VCardConfig` (10 edges) — vCard/contact module config

**Key architectural clusters (hyperedges):**
- AR Viewer runtime: A-Frame + MindAR + `.mind` file
- Tracking stability: MindAR params + One Euro Filter + target image quality
- Freemium gate: `users.plan` + RLS + `panels.enabled`
- Next.js AR integration: `ssr:false` dynamic import + COOP/COEP + bundle splitting
- Mobile AR perf: DPR cap + `MeshBasicMaterial` + geometry hoisting

**Surprising connections found:**
- `card1.jpg` → AR Target Image Design Guide → **Symmetry warning** — the e-cell MIT card may have tracking issues if it has symmetrical layout
- Gallery images in `bizzar-proto/public/gallery/` are **SVG placeholders** disguised as JPGs — not real photos
- Landing page screenshot shows a Next.js server error (module resolution failure) — needs fixing

**Top knowledge base communities:**
- Community 5 (AR Business Card UX): 8th Wall shutdown 2026, iOS WebXR = zero support, `Add to Contacts` = #1 conversion action, KPM >300 = good target
- Community 3 (AR Performance): `frameloop="always"` mandatory, `dpr={[1,1.5]}`, `antialias:false`, geometry hoisting

---

## Key Technical Decisions

- **AR viewer** (`/ar/[userId]/[cardId]`) — standalone HTML, no React shell. 8th Wall + A-Frame via script tags. **Proto uses react-three-mind (R3F + MindAR)** — migration to this for the SaaS viewer is planned.
- **3D editor** — orthographic camera (zoom 45, position [5,5,5]). Do NOT use `distanceFactor` on drei `Html` with orthographic cameras — causes massive scaling.
- **drei Html in transform mode** — use `eventSource` + `eventPrefix="client"` on Canvas + `pointer-events: none` CSS on Html wrappers to prevent blocking R3F raycasting.
- **Panel slots:** `overlay`, `top`, `left`, `right` — transforms defined in `hub-scene.tsx:PANEL_TRANSFORMS`
- **Module drag:** @dnd-kit/core — tray→grid (new placement) and grid→grid (reorder)

---

## Stack

| Layer | Tech |
|---|---|
| Frontend | Next.js 14, TypeScript, Tailwind CSS |
| 3D Editor | React Three Fiber v8 + @react-three/drei v9 |
| AR Runtime (SaaS viewer) | 8th Wall OSS Image Targets + A-Frame |
| AR Runtime (proto) | react-three-mind + MindAR (R3F-native) |
| State | Zustand |
| Backend | Supabase (Postgres, Auth, Storage) |
| Payments | Stripe |
| Drag & Drop | @dnd-kit/core |
