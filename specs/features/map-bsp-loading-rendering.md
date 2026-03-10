# Feature: Map/BSP Loading and Rendering

> Reverse-engineered from `Quake/WinQuake/model.c`, `Quake/WinQuake/model.h`, `Quake/WinQuake/bspfile.h`

---

## Overview

The BSP (Binary Space Partitioning) system is the foundation of Quake's 3D world representation. Maps are compiled offline into BSP files containing a spatial hierarchy, precomputed visibility data, lightmaps, and collision geometry. The engine loads and renders these efficiently using the BSP tree for visibility determination and the PVS (Potentially Visible Set) for culling.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/WinQuake/model.c` | Model loading: BSP, alias (MDL), sprite (SPR) |
| `Quake/WinQuake/model.h` | Model data structures |
| `Quake/WinQuake/gl_model.c` | OpenGL-specific model loading |
| `Quake/WinQuake/gl_model.h` | GL model structures |
| `Quake/WinQuake/bspfile.h` | BSP file format definition (v29) |
| `Quake/WinQuake/r_bsp.c` | BSP tree rendering traversal |
| `Quake/WinQuake/world.c` | Spatial queries, collision detection |

---

## BSP File Format (Version 29)

Defined in `Quake/WinQuake/bspfile.h`:

### File Header
```c
typedef struct {
    int version;            // Must be 29 (BSPVERSION)
    lump_t lumps[HEADER_LUMPS]; // 15 data sections
} dheader_t;
```

### 15 Data Lumps

| # | Lump | Content |
|---|------|---------|
| 0 | `LUMP_ENTITIES` | Entity definition strings (key-value pairs) |
| 1 | `LUMP_PLANES` | BSP splitting planes (normal + distance) |
| 2 | `LUMP_TEXTURES` | Mip-mapped texture data |
| 3 | `LUMP_VERTEXES` | Vertex coordinates (x, y, z floats) |
| 4 | `LUMP_VISIBILITY` | PVS compressed bitfield data |
| 5 | `LUMP_NODES` | BSP interior nodes |
| 6 | `LUMP_TEXINFO` | Texture alignment/projection info |
| 7 | `LUMP_FACES` | Surface polygons |
| 8 | `LUMP_LIGHTING` | Lightmap data (per-surface) |
| 9 | `LUMP_CLIPNODES` | Collision hull nodes |
| 10 | `LUMP_LEAFS` | BSP leaf nodes |
| 11 | `LUMP_MARKSURFACES` | Leaf-to-surface index mapping |
| 12 | `LUMP_EDGES` | Edge definitions (vertex pairs) |
| 13 | `LUMP_SURFEDGES` | Surface-to-edge index mapping |
| 14 | `LUMP_MODELS` | Brush model list (world + submodels) |

---

## Model Types

### Brush Models (BSP)
- World geometry (walls, floors, ceilings)
- Sub-models (doors, platforms, triggers)
- Loaded by `Mod_LoadBrushModel()` in `Quake/WinQuake/model.c`

### Alias Models (MDL v6)
- Characters, items, weapons, projectiles
- Animated with per-vertex keyframes
- Magic: `IDPO`, Version: 6
- Loaded by `Mod_LoadAliasModel()` in `Quake/WinQuake/model.c`

### Sprite Models (SPR v1)
- Billboard particles and effects
- Single or multi-frame sprites
- Magic: `IDSP`, Version: 1
- Loaded by `Mod_LoadSpriteModel()` in `Quake/WinQuake/model.c`

---

## BSP Loading Pipeline (`Quake/WinQuake/model.c`)

```
Mod_LoadBrushModel()
├── Mod_LoadVertexes()       — Load vertex coordinates [line ~555]
├── Mod_LoadEdges()          — Load edge definitions [line ~619]
├── Mod_LoadSurfedges()      — Load surface-to-edge mapping
├── Mod_LoadTextures()       — Load mip-mapped textures [line ~355]
├── Mod_LoadLighting()       — Load lightmap data [line ~504]
├── Mod_LoadPlanes()         — Load splitting planes [line ~1087]
├── Mod_LoadTexinfo()        — Load texture alignment [line ~646]
├── Mod_LoadFaces()          — Load surface polygons [line ~766]
├── Mod_LoadMarksurfaces()   — Load leaf-surface mapping
├── Mod_LoadVisibility()     — Load PVS data [line ~521]
├── Mod_LoadLeafs()          — Load BSP leaves [line ~897]
├── Mod_LoadNodes()          — Load BSP nodes [line ~850]
├── Mod_LoadClipnodes()      — Load collision hull [line ~944]
├── Mod_LoadSubmodels()      — Load brush sub-models
└── Mod_MakeHull0()          — Build collision hull from render nodes
```

---

## PVS (Potentially Visible Set)

- Precomputed at map compile time using `vis` tool
- Stored compressed in `LUMP_VISIBILITY`
- Each BSP leaf has a PVS bitfield indicating which other leaves are potentially visible
- `Mod_LeafPVS()` in `Quake/WinQuake/model.c` decompresses PVS for a given leaf
- Used by both renderer (skip invisible surfaces) and server (skip invisible entities for network)

### PVS Compression
- Run-length encoded: zero bytes represent spans of invisible leaves
- Decompressed to full bitfield for each leaf check
- Significant memory savings (PVS for a typical map: ~50KB compressed vs. ~500KB uncompressed)

---

## Collision Detection

### Collision Hulls
BSP files contain multiple collision hulls for different entity sizes:

| Hull | Size | Usage |
|------|------|-------|
| Hull 0 | Point | Traces, bullets |
| Hull 1 | 32×32×56 | Player standing |
| Hull 2 | 64×64×88 | Large entities (shambler) |

### Key Functions (`Quake/WinQuake/world.c`)
- `SV_Move()` — Trace a bounding box through the world
- `SV_RecursiveHullCheck()` — Recursive BSP collision test
- `SV_HullForEntity()` — Get collision hull for an entity
- `SV_LinkEdict()` — Insert entity into world spatial grid
- `SV_PointContents()` — Test what type of brush (solid, water, lava) is at a point

---

## Model Caching

```c
#define MAX_MOD_KNOWN 256
model_t mod_known[MAX_MOD_KNOWN];
int mod_numknown;
```

- `Mod_ForName()` — Load model by name, return cached pointer
- `Mod_Extradata()` — Retrieve model data from cache
- `Mod_ClearAll()` — Flush cache on level change
- `Mod_TouchModel()` — Prevent cache eviction

Models are stored in **hunk memory** (long-lived, level-scoped).

---

## Limitations

- BSP version 29 only (no BSP2 or modern formats)
- Maximum limits: 32767 nodes, 8192 leaves, 65535 faces, 65535 edges
- Single-hull collision (no per-vertex collision)
- No real-time BSP modification (static geometry only)
- PVS computation is offline only (requires map recompile for changes)
- Lightmaps are static (dynamic lights are additive overlays, not recomputed)
- No LOD (Level of Detail) for BSP geometry
