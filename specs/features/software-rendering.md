# Feature: Software Rendering Pipeline

> Reverse-engineered from `legacy-src/desktop-engine/`

---

## Overview

The software renderer implements a complete 3D rendering pipeline using CPU-only computation. It uses **edge-based scan conversion** with **BSP tree traversal** for visibility determination and **affine texture mapping** with **mip-mapping** for surface rendering.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `legacy-src/desktop-engine/r_main.c` | Rendering entry point, frustum setup, entity dispatch |
| `legacy-src/desktop-engine/r_bsp.c` | BSP tree traversal, visible surface enumeration |
| `legacy-src/desktop-engine/r_edge.c` | Edge-based scan conversion and sorting |
| `legacy-src/desktop-engine/r_surf.c` | Surface rendering with lightmaps |
| `legacy-src/desktop-engine/r_alias.c` | Alias model (MDL) rendering for characters and items |
| `legacy-src/desktop-engine/r_sprite.c` | Sprite rendering (billboard effects) |
| `legacy-src/desktop-engine/r_sky.c` | Scrolling sky texture with parallax |
| `legacy-src/desktop-engine/r_light.c` | Dynamic light contribution to lightmaps |
| `legacy-src/desktop-engine/r_part.c` | Point-based particle effects |
| `legacy-src/desktop-engine/r_misc.c` | Miscellaneous rendering utilities |
| `legacy-src/desktop-engine/r_draw.c` | 2D drawing (HUD, status bar) |
| `legacy-src/desktop-engine/d_edge.c` | Low-level edge processing |
| `legacy-src/desktop-engine/d_scan.c` | Affine texture mapping span rasterization |
| `legacy-src/desktop-engine/d_surf.c` | Surface cache management |
| `legacy-src/desktop-engine/d_init.c` | Software rasterizer initialization |
| `legacy-src/desktop-engine/d_sky.c` | Sky dome rendering |
| `legacy-src/desktop-engine/d_sprite.c` | Sprite rasterization |
| `legacy-src/desktop-engine/d_part.c` | Particle rasterization |
| `legacy-src/desktop-engine/d_fill.c` | Span filling |
| `legacy-src/desktop-engine/d_zpoint.c` | Z-buffer point operations |
| `legacy-src/desktop-engine/d_modech.c` | Mode change handling |
| `legacy-src/desktop-engine/d_polyse.c` | Polygon setup for edge rasterization |
| `legacy-src/desktop-engine/d_vars.c` | Software rasterizer global variables |
| `legacy-src/desktop-engine/draw.c` | 2D picture/graphic drawing |
| `legacy-src/desktop-engine/screen.c` | Screen management, viewport, SCR_UpdateScreen |
| `legacy-src/desktop-engine/view.c` | View calculations (bob, roll, blend) |
| `legacy-src/desktop-engine/sbar.c` | Status bar (HUD) rendering |

---

## Rendering Pipeline

```
R_RenderView() [r_main.c]
├── R_SetupFrame()         — Calculate view position, build frustum planes
├── R_MarkLeaves()         — Mark visible BSP leaves using PVS
├── R_PushDlights()        — Push dynamic lights into BSP
├── R_EdgeDrawing()        — Begin edge-based rasterization
│   └── R_RenderWorld()    — Traverse BSP tree [r_bsp.c]
│       └── R_RecursiveWorldNode() — Recursive front-to-back traversal
│           └── R_RenderFace()     — Generate edges for each visible surface
├── R_DrawBEntitiesOnList() — Render brush model entities (doors, platforms)
├── R_DrawEntitiesOnList()  — Render alias models (players, items)
│   └── R_AliasDrawModel() [r_alias.c] — Per-model rendering
├── R_DrawViewModel()       — Render first-person weapon model
├── R_DrawParticles()       — Render particle effects
└── R_DrawWaterSurfaces()   — Render translucent water
```

---

## Key Techniques

### PVS (Potentially Visible Set)
- Precomputed at map compile time, stored in `LUMP_VISIBILITY` of BSP file
- Compressed bitfield: each leaf has a bit for every other leaf indicating mutual visibility
- `Mod_LeafPVS()` in `legacy-src/desktop-engine/model.c` decompresses PVS for current leaf
- `R_MarkLeaves()` marks all visible leaves, skipping entire BSP subtrees

### Lightmaps
- Precomputed lightmaps stored in `LUMP_LIGHTING` of BSP file
- Dynamic lights add to base lightmaps via `R_AddDynamicLights()` in `legacy-src/desktop-engine/r_light.c`
- Combined lightmap + texture cached in surface cache (`d_surf.c`)

### Mip-Mapping
- Textures stored at 4 mip levels (1:1, 1:2, 1:4, 1:8)
- Mip level selected based on surface distance from camera
- Reduces aliasing and improves performance at distance

### Surface Cache
- `D_CacheSurface()` in `legacy-src/desktop-engine/d_surf.c` caches pre-lit textured surfaces
- Cache entries reused across frames when lighting hasn't changed
- Evicted when cache is full (LRU policy)

---

## Limitations

- No hardware acceleration — CPU-only
- 8-bit color depth (256-color palette)
- No texture filtering (point sampling only)
- No alpha blending (binary transparency only)
- x86 assembly required for acceptable performance on target hardware
- Single-threaded — cannot leverage multiple CPU cores
