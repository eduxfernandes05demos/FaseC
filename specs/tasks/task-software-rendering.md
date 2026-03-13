# Task: Software Rendering Subsystem

> Technical implementation specification with file references.

---

## Subsystem Overview

The software rendering subsystem implements CPU-based 3D rendering using edge-based scan conversion, BSP tree visibility, and affine texture mapping.

---

## File Inventory

### Rendering Core (`legacy-src/desktop-engine/`)

| File | LOC (approx.) | Responsibility |
|------|---------------|----------------|
| `r_main.c` | 1,000 | Entry point, frustum, entity dispatch |
| `r_bsp.c` | 650 | BSP tree front-to-back traversal |
| `r_edge.c` | 800 | Edge sorting and scan conversion |
| `r_surf.c` | 800 | Surface rendering with lightmaps |
| `r_alias.c` | 700 | Alias model (MDL) rendering |
| `r_sprite.c` | 300 | Sprite billboard rendering |
| `r_sky.c` | 200 | Scrolling sky texture |
| `r_light.c` | 400 | Dynamic light application |
| `r_part.c` | 300 | Particle effects |
| `r_draw.c` | 200 | 2D drawing primitives |
| `r_misc.c` | 200 | Miscellaneous rendering helpers |
| `r_aclip.c` | 200 | Alias model clipping |
| `r_efrag.c` | 200 | Entity fragment management |

### Software Rasterizer (`legacy-src/desktop-engine/`)

| File | Responsibility |
|------|----------------|
| `d_edge.c` | Edge processing and span generation |
| `d_scan.c` | Affine texture mapping scan lines |
| `d_surf.c` | Surface cache management |
| `d_init.c` | Rasterizer initialization |
| `d_sky.c` | Sky dome rendering |
| `d_sprite.c` | Sprite rasterization |
| `d_part.c` | Particle rasterization |
| `d_fill.c` | Span color filling |
| `d_zpoint.c` | Z-buffer point operations |
| `d_modech.c` | Video mode change handling |
| `d_polyse.c` | Polygon edge setup |
| `d_vars.c` | Global rasterizer variables |

### x86 Assembly Optimizations (`legacy-src/desktop-engine/`)

| File | Optimizes |
|------|-----------|
| `d_draw.s` | Drawing inner loops |
| `d_draw16.s` | 16-pixel span drawing |
| `d_parta.s` | Particle rendering |
| `d_polysa.s` | Polygon setup |
| `d_scana.s` | Scan line rendering |
| `d_varsa.s` | Rasterizer variable access |
| `d_copy.s` | Memory copy |
| `r_aclipa.s` | Alias model clipping |
| `r_aliasa.s` | Alias model rendering |
| `r_drawa.s` | Drawing routines |
| `r_edgea.s` | Edge processing |
| `r_varsa.s` | Renderer variable access |
| `surf8.s` | 8-bit surface rendering |
| `surf16.s` | 16-bit surface rendering |

### Headers

| File | Content |
|------|---------|
| `r_local.h` | Software renderer internal definitions |
| `r_shared.h` | Shared renderer definitions |
| `render.h` | Public renderer interface |
| `d_iface.h` | Rasterizer interface |
| `d_ifacea.h` | Assembly rasterizer interface |
| `d_local.h` | Rasterizer internal definitions |

### Screen and View

| File | Responsibility |
|------|----------------|
| `screen.c` / `screen.h` | Screen management, `SCR_UpdateScreen()`, viewport |
| `view.c` / `view.h` | View calculations (bob, roll, damage blend, underwater warp) |
| `sbar.c` / `sbar.h` | Status bar / HUD rendering |
| `draw.c` / `draw.h` | 2D picture loading and drawing |

---

## Key Algorithms

### BSP Traversal (`r_bsp.c`)
- `R_RecursiveWorldNode()` — Front-to-back recursive traversal
- Uses dot product with split plane to determine which side camera is on
- Visits near side first, then far side
- Generates edge lists for visible surfaces

### Edge-Based Scan Conversion (`r_edge.c`)
- Surfaces are decomposed into edges
- Edges are sorted by Y coordinate
- Active Edge Table (AET) maintained per scan line
- Correct visibility via depth comparison at each pixel

### Surface Cache (`d_surf.c`)
- Pre-lit + textured surfaces cached for reuse
- `D_CacheSurface()` checks if surface needs re-rendering
- Cache invalidated when dynamic lights change
- LRU eviction when cache is full

### Lightmap Combination
- Base lightmap from BSP + dynamic light contribution
- `R_BuildLightMap()` in `r_light.c` combines both
- Result cached in surface cache entry

---

## Data Flow

```
BSP Tree → R_RecursiveWorldNode() → Edge Lists → R_ScanEdges()
    → Span Generation → D_DrawSpans8() → Framebuffer
    
Alias Model → R_AliasDrawModel() → Transform → Clip → Rasterize → Framebuffer

Particle → R_DrawParticles() → D_DrawParticle() → Framebuffer
```

---

## Performance Considerations

- x86 assembly inner loops critical for acceptable performance
- Surface cache avoids redundant lightmap + texture computation
- PVS culling eliminates ~70-90% of world geometry
- Mip-mapping reduces pixel fill at distance
- Edge sorting provides exact visibility (no overdraw)
