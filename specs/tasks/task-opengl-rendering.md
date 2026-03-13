# Task: OpenGL Rendering Subsystem

> Technical implementation specification with file references.

---

## Subsystem Overview

The OpenGL renderer provides hardware-accelerated 3D rendering as an alternative to the software renderer. It targets OpenGL 1.1 fixed-function pipeline and is conditionally compiled with the `GLQUAKE` preprocessor macro.

---

## File Inventory (`legacy-src/desktop-engine/`)

| File | Responsibility |
|------|----------------|
| `gl_rmain.c` | GL rendering entry point, entity rendering, setup |
| `gl_rsurf.c` | GL BSP surface/world rendering, lightmap management |
| `gl_mesh.c` | GL alias model display list compilation |
| `gl_rlight.c` | GL dynamic lighting (additive spheres) |
| `gl_draw.c` | GL 2D drawing, texture upload (palette → RGBA) |
| `gl_warp.c` | GL sky and water surface warping |
| `gl_model.c` | GL-specific model loading with texture creation |
| `gl_model.h` | GL model data structures |
| `gl_refrag.c` | GL entity fragment storage |
| `gl_rmisc.c` | GL miscellaneous rendering utilities |
| `gl_screen.c` | GL screen management and updates |
| `gl_test.c` | GL testing/debugging tools |
| `gl_vidnt.c` | GL Windows WGL video context |
| `gl_vidlinux.c` | GL Linux SVGA video |
| `gl_vidlinuxglx.c` | GL Linux X11/GLX video |
| `glquake.h` | GL-specific definitions, OpenGL includes |
| `glquake2.h` | Additional GL definitions |
| `gl_warp_sin.h` | Precomputed sine table for water warping |

---

## Key Implementation Details

### Texture Management
- `GL_Upload8()` — Convert 8-bit palette texture to 32-bit RGBA and upload via `glTexImage2D()`
- `GL_Upload32()` — Upload 32-bit RGBA texture
- Textures bound with `glBindTexture()` for efficient switching
- Palette lookup at upload time (not per-pixel at runtime)

### Lightmap System
- Lightmaps uploaded as separate GL textures
- Applied via multi-pass or multitexture blending
- Dynamic light changes trigger partial `glTexSubImage2D()` updates
- More efficient than software renderer's surface cache for frequently-changing lights

### Alias Model Display Lists
- `GL_MakeAliasModelDisplayLists()` — Pre-compile model geometry into GL display lists
- Frame interpolation done in software, submitted via immediate mode
- Significant performance win for repeated model rendering

### Water and Sky
- Water surfaces use `EmitWaterPolys()` with sine-based vertex warping
- Sky uses either sky box (6 textures) or scrolling parallax sky texture
- Both use alpha blending for translucency

---

## Build Configuration

- Compile-time selection via `GLQUAKE` define
- Links against: `opengl32.lib` (Windows), `-lMesaGL` or `-lGL` (Linux)
- Include paths: Mesa3D, system OpenGL headers
- Separate build targets: `glquake`, `glquake.glx`, `glquake.3dfxgl`, `glqwcl`
