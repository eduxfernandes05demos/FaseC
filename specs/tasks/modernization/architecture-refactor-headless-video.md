# Task: Architecture Refactor — Headless Video Output

> Extract rendering into a pluggable module, create headless video output for server/streaming.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P1 — Modularization |
| **Estimated Complexity** | High |
| **Estimated Effort** | 2 weeks |
| **Dependencies** | Build system (CMake), Security remediation |
| **Blocks** | Containerization, Streaming gateway |
| **Phase** | Phase 2 — Modularization |

---

## Objective

Create a headless video driver that renders frames to an in-memory buffer without requiring display hardware. This enables the engine to run in containers and cloud environments without X11, Wayland, or any display server.

---

## Current Video Architecture

The engine has platform-specific video drivers selected at compile time:

| File | Platform | Key Functions |
|------|----------|---------------|
| `legacy-src/desktop-engine/vid_win.c` | Windows | `VID_Init()`, `VID_Update()`, `VID_AllocBuffers()` (line 260) |
| `legacy-src/desktop-engine/vid_svgalib.c` | Linux SVGAlib | `VID_Init()` (line 555), `VID_Update()` (line 722) |
| `legacy-src/desktop-engine/vid_x.c` | Linux X11 | `VID_Init()`, `VID_Update()` |
| `legacy-src/desktop-engine/vid_dos.c` | DOS | `VID_Init()`, `VID_Update()` |
| `legacy-src/desktop-engine/vid_null.c` | Stub | Minimal stubs (exists but incomplete) |

### Current Video Interface

The engine accesses video through global `viddef_t vid` (defined in `legacy-src/desktop-engine/vid.h`):

```c
typedef struct {
    pixel_t     *buffer;          /* Pixel buffer for rendering */
    pixel_t     *colormap;        /* Color lookup table */
    unsigned short *colormap16;
    int         fullbright;
    unsigned    rowbytes;          /* Bytes per row */
    unsigned    width;
    unsigned    height;
    float       aspect;
    int         numpages;
    int         recalc_refdef;
    pixel_t     *conbuffer;
    int         conrowbytes;
    unsigned    conwidth;
    unsigned    conheight;
    int         maxwarpwidth;
    int         maxwarpheight;
    pixel_t     *direct;
} viddef_t;
```

### Critical Dependency: VID_AllocBuffers

`legacy-src/desktop-engine/vid_win.c:260` — `VID_AllocBuffers()` allocates the Z-buffer and surface cache from Hunk memory. The headless driver MUST replicate this:

```c
/* From vid_win.c:260 */
void VID_AllocBuffers(int width, int height) {
    /* Allocates d_pzbuffer (Z-buffer) */
    /* Allocates surface cache via D_InitCaches */
}
```

---

## Implementation Steps

### Step 1: Define Video Interface

**New file**: `legacy-src/desktop-engine/video_interface.h`

```c
typedef struct video_interface_s {
    const char *name;
    void (*init)(unsigned char *palette);
    void (*shutdown)(void);
    void (*update)(vrect_t *rects);
    void (*set_palette)(unsigned char *palette);
    void (*set_mode)(int width, int height);

    /* New: Framebuffer access for streaming */
    const pixel_t *(*get_framebuffer)(int *width, int *height);
} video_interface_t;
```

### Step 2: Create Headless Video Driver

**New file**: `legacy-src/desktop-engine/vid_headless.c`

Must implement:

| Function | Purpose | Reference |
|----------|---------|-----------|
| `VID_Init()` | Allocate framebuffer + Z-buffer + surface cache | `vid_svgalib.c:555` |
| `VID_Shutdown()` | Free allocated buffers | `vid_svgalib.c` |
| `VID_Update()` | No-op (frame stays in buffer) | `vid_svgalib.c:722` |
| `VID_SetPalette()` | Store palette for color mapping | `vid_svgalib.c:447` |
| `VID_AllocBuffers()` | Allocate Z-buffer and surface cache | `vid_win.c:260` |
| `VID_GetFramebuffer()` | **New**: Return pointer to rendered frame | — |

**Critical**: Must set up `vid.buffer`, `vid.width`, `vid.height`, `vid.rowbytes`, `d_pzbuffer` (Z-buffer) for the software renderer to function.

### Step 3: Wire Into Engine Initialization

Modify `legacy-src/desktop-engine/host.c:835` (`Host_Init`) to select video driver based on configuration:

```c
/* In Host_Init: */
if (headless_mode) {
    VID_InitHeadless(palette);
} else {
    VID_Init(palette);  /* Original behavior */
}
```

### Step 4: Add Framebuffer Extraction API

**In `vid_headless.c`**:

```c
const pixel_t *VID_GetFramebuffer(int *width, int *height) {
    *width = vid.width;
    *height = vid.height;
    return vid.buffer;
}
```

This API is called by the streaming gateway to extract rendered frames.

### Step 5: Handle Screen Management

`legacy-src/desktop-engine/screen.c` calls `VID_Update()` at the end of `SCR_UpdateScreen()`. In headless mode, this must:
- Complete all rendering to `vid.buffer`
- Signal frame availability (for streaming)
- Not attempt any display operations

---

## Files Affected

| File | Change Type | Description |
|------|------------|-------------|
| `legacy-src/desktop-engine/vid_headless.c` | **New** | Headless video driver |
| `legacy-src/desktop-engine/video_interface.h` | **New** | Video interface definition |
| `legacy-src/desktop-engine/vid.h` | Minor edit | Add `VID_GetFramebuffer()` declaration |
| `legacy-src/desktop-engine/host.c` | Minor edit | Add headless mode startup logic (line 835) |
| `legacy-src/desktop-engine/screen.c` | Minor edit | Handle headless mode in `SCR_UpdateScreen()` |
| `legacy-src/desktop-engine/CMakeLists.txt` | Edit | Add `vid_headless.c` to headless build target |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_headless_init` | Engine starts in headless mode without crash |
| `test_framebuffer_valid` | `VID_GetFramebuffer()` returns non-NULL pointer with correct dimensions |
| `test_framebuffer_content` | After rendering, framebuffer contains non-zero (non-black) pixels |
| `test_headless_shutdown` | Engine shuts down cleanly in headless mode |
| `test_headless_server` | Headless engine runs server, QW client connects successfully |
| `test_headless_timedemo` | Timedemo completes in headless mode without crash |

---

## Acceptance Criteria

- [ ] Engine starts with `--headless` flag on a machine without X11/display
- [ ] Software renderer writes frames to in-memory buffer
- [ ] `VID_GetFramebuffer()` returns valid pixel data
- [ ] Z-buffer (`d_pzbuffer`) is properly allocated
- [ ] Surface cache is properly initialized via `D_InitCaches`
- [ ] Server simulation runs correctly in headless mode
- [ ] QW client can connect to headless server and play
- [ ] Memory usage in headless mode ≤ 64 MB
- [ ] No display-related error messages or crashes
- [ ] Timedemo completes without rendering artifacts (verified via frame dump)
