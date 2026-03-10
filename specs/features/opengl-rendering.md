# Feature: OpenGL Rendering Pipeline

> Reverse-engineered from `Quake/WinQuake/gl_*.c`

---

## Overview

The OpenGL renderer replaces the software rasterizer with hardware-accelerated polygon rendering while reusing the BSP traversal and visibility systems. It targets OpenGL 1.1 and supports multiple backends including Windows WGL, Linux Mesa3D, Linux GLX, and 3Dfx Voodoo MiniGL.

Enabled by the `GLQUAKE` preprocessor macro at compile time.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/WinQuake/gl_rmain.c` | GL rendering entry point, entity rendering |
| `Quake/WinQuake/gl_rsurf.c` | GL BSP surface/world rendering |
| `Quake/WinQuake/gl_mesh.c` | GL alias model mesh compilation and rendering |
| `Quake/WinQuake/gl_rlight.c` | GL dynamic lighting (point lights as spheres) |
| `Quake/WinQuake/gl_draw.c` | GL 2D drawing, texture upload (palette → RGBA) |
| `Quake/WinQuake/gl_warp.c` | GL sky and water surface warping |
| `Quake/WinQuake/gl_model.c` | GL-specific model loading (texture creation) |
| `Quake/WinQuake/gl_model.h` | GL model data structures |
| `Quake/WinQuake/gl_refrag.c` | GL entity fragment storage |
| `Quake/WinQuake/gl_rmisc.c` | GL miscellaneous rendering utilities |
| `Quake/WinQuake/gl_screen.c` | GL screen management |
| `Quake/WinQuake/gl_test.c` | GL testing/debugging utilities |
| `Quake/WinQuake/gl_vidnt.c` | GL Windows video driver (WGL context) |
| `Quake/WinQuake/gl_vidlinux.c` | GL Linux SVGA video driver |
| `Quake/WinQuake/gl_vidlinuxglx.c` | GL Linux X11/GLX video driver |
| `Quake/WinQuake/glquake.h` | GL-specific definitions and headers |

---

## Key Differences from Software Renderer

| Aspect | Software | OpenGL |
|--------|----------|--------|
| Rasterization | CPU edge-based scan conversion | GPU polygon rasterization |
| Texture mapping | Affine (CPU) | Perspective-correct (GPU) |
| Color depth | 8-bit (256 palette) | 32-bit RGBA |
| Lightmaps | Pre-lit surface cache | Separate lightmap textures, multitexture |
| Dynamic lights | Modify lightmap, invalidate cache | Rendered as additive spheres |
| Transparency | Binary (on/off) | Alpha blending |
| Resolution | Limited by CPU performance | Limited by GPU fillrate |

---

## Rendering Pipeline

```
R_RenderView() [gl_rmain.c]
├── R_SetupFrame()          — View parameters
├── R_SetFrustum()          — Calculate frustum planes
├── R_MarkLeaves()          — PVS visibility marking
├── R_DrawWorld()            — BSP world surfaces [gl_rsurf.c]
│   ├── Opaque surfaces     — Standard textured polygons
│   ├── Lightmap passes     — Multitexture or multi-pass
│   └── Sky surfaces        — Sky box or scrolling sky
├── R_DrawEntitiesOnList()  — Alias models + brush entities
│   └── R_DrawAliasModel()  — Compiled display lists [gl_mesh.c]
├── R_RenderDlights()       — Dynamic light spheres [gl_rlight.c]
├── R_DrawParticles()       — Point particles
└── R_DrawWaterSurfaces()   — Translucent water with warping
```

---

## Key Techniques

### Texture Upload
- Palette-indexed 8-bit textures converted to RGBA on upload
- `GL_Upload8()` / `GL_Upload32()` in `Quake/WinQuake/gl_draw.c`
- Textures bound to OpenGL texture objects with `glBindTexture()`

### Lightmap Textures
- Lightmaps uploaded as separate textures
- Blended with surface textures via multitexture or multi-pass rendering
- Dynamic light changes trigger partial lightmap texture re-upload

### Alias Model Display Lists
- `GL_MakeAliasModelDisplayLists()` in `Quake/WinQuake/gl_mesh.c`
- Compiled into OpenGL display lists for efficient repeated rendering
- Vertex animation interpolated between frames

---

## Limitations

- OpenGL 1.1 only — no shaders, no VBOs, no FBOs
- Fixed-function pipeline
- No anisotropic filtering
- No anti-aliasing (MSAA)
- Palette-to-RGBA conversion loses some color precision
- Dynamic light rendering is approximate (additive spheres, not shadows)
