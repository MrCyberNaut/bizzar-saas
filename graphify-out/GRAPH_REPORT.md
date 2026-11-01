# Graph Report - d:/Projects/BizzAR-SaaS  (2026-05-20)

## Corpus Check
- 100 files · ~53,862 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 309 nodes · 435 edges · 31 communities (17 shown, 14 thin omitted)
- Extraction: 94% EXTRACTED · 6% INFERRED · 0% AMBIGUOUS · INFERRED: 25 edges (avg confidence: 0.81)
- Token cost: 0 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_panel-editor.tsx|panel-editor.tsx]]
- [[_COMMUNITY_CardAnchor.tsx|CardAnchor.tsx]]
- [[_COMMUNITY_MindAR.js Image Tracking Library|MindAR.js Image Tracking Library]]
- [[_COMMUNITY_AR Target Image Design Guide|AR Target Image Design Guide]]
- [[_COMMUNITY_ModuleConfig|ModuleConfig]]
- [[_COMMUNITY_AR Business Card UX Patterns|AR Business Card UX Patterns]]
- [[_COMMUNITY_WebAR Tracking Knowledge Base Index|WebAR Tracking Knowledge Base Index]]
- [[_COMMUNITY_page.tsx|page.tsx]]
- [[_COMMUNITY_layout.tsx|layout.tsx]]
- [[_COMMUNITY_route.ts|route.ts]]
- [[_COMMUNITY_Phase 8 Module System|Phase 8: Module System]]
- [[_COMMUNITY_config|config]]
- [[_COMMUNITY_mind-ar.d.ts|mind-ar.d.ts]]
- [[_COMMUNITY_Supabase Data Model (userscardspanelsmodules)|Supabase Data Model (users/cards/panels/modules)]]
- [[_COMMUNITY_8thwall.d.ts|8thwall.d.ts]]
- [[_COMMUNITY_WebXR Image Tracking Viability (2026)|WebXR Image Tracking Viability (2026)]]
- [[_COMMUNITY_webhook.ts|webhook.ts]]
- [[_COMMUNITY_bizzar App CLAUDE.md (Graphify Nav Instructions)|bizzar App CLAUDE.md (Graphify Nav Instructions)]]
- [[_COMMUNITY_bizzar Next.js App README|bizzar Next.js App README]]
- [[_COMMUNITY_Next.js 14 (App Router)|Next.js 14 (App Router)]]
- [[_COMMUNITY_Zustand State Management|Zustand State Management]]
- [[_COMMUNITY_Route dashboard|Route: /dashboard]]
- [[_COMMUNITY_Phase 1 Project Scaffold & Config|Phase 1: Project Scaffold & Config]]
- [[_COMMUNITY_Phase 3 Auth & Dashboard Shell|Phase 3: Auth & Dashboard Shell]]
- [[_COMMUNITY_Phase 4 Card Creation & Image Upload|Phase 4: Card Creation & Image Upload]]
- [[_COMMUNITY_Phase 11 Publishing, QR Code & Sharing|Phase 11: Publishing, QR Code & Sharing]]
- [[_COMMUNITY_Phase 12 Landing Page & Pricing|Phase 12: Landing Page & Pricing]]
- [[_COMMUNITY_card1.jpg - Sample AR Business Card Target|card1.jpg - Sample AR Business Card Target]]

## God Nodes (most connected - your core abstractions)
1. `ModuleConfig` - 17 edges
2. `EditorClient()` - 16 edges
3. `Database` - 10 edges
4. `VCardConfig` - 10 edges
5. `WebAR Tracking Knowledge Base Index` - 9 edges
6. `DashboardClient()` - 8 edges
7. `Phase 8: Module System` - 8 edges
8. `NewCardModal()` - 7 edges
9. `createClient()` - 7 edges
10. `Panel()` - 7 edges

## Surprising Connections (you probably didn't know these)
- `Gallery Images - SVG Placeholders (1-4.jpg)` --conceptually_related_to--> `@react-three/drei AR Patterns`  [AMBIGUOUS]
  bizzar-proto/public/gallery/1.jpg → knowledge/10-r3f-drei-ar-patterns.md
- `Sample AR Business Card - Arnav Sawant e-cell MIT Manipal` --conceptually_related_to--> `AR Target Image Design Guide`  [INFERRED]
  bizzar-proto/public/card1.jpg → knowledge/17-ar-target-image-design.md
- `BizzAR Landing Page - Next.js Server Error Screenshot` --conceptually_related_to--> `Next.js + WebAR Integration Patterns`  [INFERRED]
  landing-page.png → knowledge/18-nextjs-webar-integration.md
- `Sample AR Business Card - Arnav Sawant e-cell MIT Manipal` --conceptually_related_to--> `Symmetry Causes AR Tracking Orientation Flip`  [INFERRED]
  bizzar-proto/public/card1.jpg → knowledge/17-ar-target-image-design.md
- `BizzAR Prototype README` --references--> `Two Separate Rendering Environments`  [INFERRED]
  bizzar-proto/README.md → BizzAR-master-prompt.md

## Hyperedges (group relationships)
- **AR Viewer Runtime Stack (A-Frame + 8thWall/MindAR + .mind file)** — concept_ar_viewer, concept_image_tracking, concept_mind_file, tech_aframe, tech_mindar [EXTRACTED 1.00]
- **Tracking Stability Improvement Triad (params + filter + target quality)** — concept_mindar_params, concept_one_euro_filter, concept_tracking_quality [INFERRED 0.90]
- **Freemium Gate Data Enforcement (users.plan + RLS + panels.enabled)** — concept_freemium_gate, db_users, db_panels, tech_supabase [EXTRACTED 1.00]
- **MindAR Target Quality & Tracking Pipeline** — doc_15_mindar_compiler_api, doc_16_mindar_tracking_pipeline, doc_17_ar_target_image_design [INFERRED 0.90]
- **Next.js AR Integration: SSR, COOP/COEP, Bundle Splitting** — concept_ssr_false_dynamic_import, concept_coop_coep_ar_route, concept_ar_bundle_splitting [EXTRACTED 0.95]
- **Mobile AR Performance: DPR Cap, MeshBasicMaterial, Geometry Hoisting** — concept_dpr_cap, concept_meshbasicmaterial_ar, concept_geometry_hoisting [INFERRED 0.85]

## Communities (31 total, 14 thin omitted)

### Community 0 - "panel-editor.tsx"
Cohesion: 0.06
Nodes (42): Panel(), PanelProps, EditorClient(), EditorClientProps, DashboardClient(), DashboardClientProps, CardMesh(), CardMeshProps (+34 more)

### Community 1 - "CardAnchor.tsx"
Cohesion: 0.1
Nodes (19): CardAnchor(), Props, PANEL_TRANSFORMS, ContributionDay, fetchContributions(), downloadVCF(), generateVCF(), Bio() (+11 more)

### Community 2 - "MindAR.js Image Tracking Library"
Cohesion: 0.08
Nodes (38): MindAR Browser Compiler HTML, Standalone AR Viewer (no React), drei Html + Orthographic Camera Fix, Freemium Gate (Server-side RLS), 3D Hub Editor (R3F Isometric View), 8th Wall OSS / MindAR Image Tracking, MindAR Compiler (Browser WASM), .mind File (Compiled Image Target) (+30 more)

### Community 3 - "AR Target Image Design Guide"
Cohesion: 0.07
Nodes (38): AR Vendor Bundle Splitting via Webpack, MindAR Compiler Requires SSR False, COOP/COEP Headers Scoped to AR Route, DPR Cap [1, 1.5] for Mobile AR, drei Html transform vs screen-space, Feature Distribution Across Image Quadrants, frameloop='always' for AR, Geometry Hoisting Outside Components (+30 more)

### Community 4 - "ModuleConfig"
Cohesion: 0.1
Nodes (12): ModuleConfigFormProps, AnimatedOverlayConfig, BioConfig, CalendlyConfig, FREE_MODULES, GitHubConfig, ImageGalleryConfig, ModuleConfig (+4 more)

### Community 5 - "AR Business Card UX Patterns"
Cohesion: 0.14
Nodes (19): 1280x720 Sweet Spot for MindAR, 8th Wall Shutdown and OSS Transition, Add to Contacts as Primary AR Conversion Action, AlvaAR GPL License Blocker, AR Differentiation Gap in Business Card Market, facingMode ideal vs exact, In-App Browser Detection for AR, model-viewer AR Quick Look iOS Fallback (+11 more)

### Community 6 - "WebAR Tracking Knowledge Base Index"
Cohesion: 0.18
Nodes (17): BizzAR CLAUDE.md Project Instructions, BizzAR Product Design Spec, BizzAR Error Log, BizzAR Implementation Plan, BizzAR Master Context Prompt, BizzAR Prototype README, getUserMedia Camera Constraints for WebAR, Mobile WebAR GPU Contention (+9 more)

### Community 7 - "page.tsx"
Cohesion: 0.12
Nodes (5): ARErrorBoundary, ARPageClient(), EBState, ARScene(), tiers

### Community 8 - "layout.tsx"
Cohesion: 0.22
Nodes (5): geistMono, geistSans, metadata, navItems, Sidebar()

### Community 9 - "route.ts"
Cohesion: 0.36
Nodes (9): ARPageData, buildARPage(), errorPage(), escapeHtml(), GET(), renderModule(), getServiceClient(), getStripe() (+1 more)

### Community 10 - "Phase 8: Module System"
Cohesion: 0.22
Nodes (9): Module: Animated Overlay (Pro), Module: Bio / Text Block, Module: Calendly (Pro), Module: GitHub Graph (Pro), Module: Image Gallery (Pro), Module: Social Icons, Module: vCard (Add to Contacts), Module: Video Panel (Pro) (+1 more)

### Community 12 - "mind-ar.d.ts"
Cohesion: 0.5
Nodes (3): ARAnchorProps, ARViewProps, Window

### Community 13 - "Supabase Data Model (users/cards/panels/modules)"
Cohesion: 0.5
Nodes (4): Supabase Free Tier Auto-Pause, Supabase Data Model (users/cards/panels/modules), Phase 2: Supabase Schema & RLS, Supabase (Postgres+Auth+Storage+Edge+Realtime)

### Community 15 - "WebXR Image Tracking Viability (2026)"
Cohesion: 1.0
Nodes (3): SLAM vs Image Tracking Tradeoff, WebAR Hybrid Tracking Strategy (2026), WebXR Image Tracking Viability (2026)

## Ambiguous Edges - Review These
- `@react-three/drei AR Patterns` → `Gallery Images - SVG Placeholders (1-4.jpg)`  [AMBIGUOUS]
  bizzar-proto/public/gallery/1.jpg · relation: conceptually_related_to

## Knowledge Gaps
- **97 isolated node(s):** `geistSans`, `geistMono`, `metadata`, `DashboardClientProps`, `EditorClientProps` (+92 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **14 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What is the exact relationship between `@react-three/drei AR Patterns` and `Gallery Images - SVG Placeholders (1-4.jpg)`?**
  _Edge tagged AMBIGUOUS (relation: conceptually_related_to) - confidence is low._
- **Why does `MindAR.js Image Tracking Library` connect `MindAR.js Image Tracking Library` to `AR Target Image Design Guide`?**
  _High betweenness centrality (0.050) - this node is a cross-community bridge._
- **Why does `Trackability Score for Card Upload Validation` connect `AR Target Image Design Guide` to `MindAR.js Image Tracking Library`?**
  _High betweenness centrality (0.049) - this node is a cross-community bridge._
- **Why does `MindAR Compiler (Browser WASM)` connect `MindAR.js Image Tracking Library` to `AR Target Image Design Guide`?**
  _High betweenness centrality (0.045) - this node is a cross-community bridge._
- **What connects `geistSans`, `geistMono`, `metadata` to the rest of the system?**
  _97 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `panel-editor.tsx` be split into smaller, more focused modules?**
  _Cohesion score 0.06 - nodes in this community are weakly interconnected._
- **Should `CardAnchor.tsx` be split into smaller, more focused modules?**
  _Cohesion score 0.1 - nodes in this community are weakly interconnected._