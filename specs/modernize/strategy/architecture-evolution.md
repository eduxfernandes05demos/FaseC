# Architecture Evolution Plan — Quake Engine

> Architecture improvement plan: modularization, headless mode, API layer, containerization strategy.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document defines the architecture evolution from a monolithic C engine to a modular, containerized service architecture suitable for cloud-native game streaming on Azure. The evolution preserves the core engine while wrapping it in modern service layers.

---

## 2. Target Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Infrastructure (Azure)               │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐     │
│  │ API Gateway  │  │   Session   │  │    Monitoring     │     │
│  │  (Streaming) │  │   Manager   │  │  (Azure Monitor)  │     │
│  └──────┬──────┘  └──────┬──────┘  └──────────────────┘     │
│         │                │                                    │
│  ┌──────▼──────────────▼─────────────────────────────┐      │
│  │              Container Orchestration (AKS)          │      │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │      │
│  │  │  Engine    │  │  Engine    │  │  Engine    │       │      │
│  │  │  Session 1 │  │  Session 2 │  │  Session N │       │      │
│  │  │  (headless)│  │  (headless)│  │  (headless)│       │      │
│  │  └───────────┘  └───────────┘  └───────────┘       │      │
│  └─────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Modularization Strategy

### 3.1 Module Boundaries

| Module | Current Files | New Interface | Responsibility |
|--------|--------------|---------------|---------------|
| **Core Engine** | `host.c`, `common.c`, `cmd.c`, `cvar.c`, `zone.c` | `engine_t` struct | Frame loop, configuration, memory |
| **Server** | `sv_main.c`, `sv_phys.c`, `sv_move.c`, `sv_user.c` | `server_t` struct | Game simulation, physics, QuakeC |
| **Client** | `cl_main.c`, `cl_parse.c`, `cl_input.c`, `cl_tent.c` | `client_t` struct | Input processing, prediction, state |
| **Renderer** | `r_main.c` / `gl_rmain.c` + supporting files | `renderer_interface_t` | Visual output |
| **Audio** | `snd_dma.c`, `snd_mix.c`, `snd_mem.c` | `audio_interface_t` | Sound mixing and playback |
| **Network** | `net_main.c`, `net_dgrm.c`, transport drivers | `network_interface_t` | Packet transport |
| **Input** | `in_win.c`, `in_dos.c`, `keys.c` | `input_interface_t` | User input capture |
| **Platform** | `sys_*.c`, `vid_*.c` | `platform_interface_t` | OS abstraction |

### 3.2 Interface Design Pattern

Replace direct function calls with **function pointer structs** (C vtable pattern):

```c
/* renderer_interface.h */
typedef struct renderer_interface_s {
    const char *name;
    int  (*init)(void *config);
    void (*shutdown)(void);
    void (*begin_frame)(void);
    void (*render_view)(struct refdef_s *refdef);
    void (*end_frame)(void);
    void (*get_framebuffer)(uint8_t *buffer, int *width, int *height);
} renderer_interface_t;

/* Implementations register at startup */
extern renderer_interface_t renderer_software;  /* r_main.c */
extern renderer_interface_t renderer_opengl;    /* gl_rmain.c */
extern renderer_interface_t renderer_headless;  /* r_headless.c (new) */
```

### 3.3 State Encapsulation

Replace global variables with **engine state structs**:

```c
/* Current (globals in host.c lines 36-82): */
double      host_frametime;
double      host_time;
double      realtime;
int         host_framecount;
qboolean    host_initialized;

/* Target: */
typedef struct engine_state_s {
    double      frame_time;
    double      host_time;
    double      real_time;
    int         frame_count;
    qboolean    initialized;
    /* ... */
    struct server_state_s   *server;
    struct client_state_s   *client;
    struct renderer_interface_s *renderer;
    struct audio_interface_s    *audio;
    struct network_interface_s  *network;
} engine_state_t;
```

This enables **multiple engine instances** per process (critical for session management).

---

## 4. Headless Mode Architecture

### 4.1 Headless Video Driver

**New file**: `vid_headless.c`

Purpose: Render frames to an in-memory buffer without requiring display hardware.

```c
/* vid_headless.c — Frame buffer without display */
static uint8_t *framebuffer;
static int fb_width, fb_height;

void VID_Init(unsigned char *palette) {
    fb_width = 640;   /* Configurable via env var */
    fb_height = 480;
    framebuffer = malloc(fb_width * fb_height * 4); /* RGBA */
}

void VID_Update(vrect_t *rects) {
    /* No-op: frame stays in buffer for extraction */
}

/* New API for streaming pipeline */
const uint8_t *VID_GetFramebuffer(int *width, int *height) {
    *width = fb_width;
    *height = fb_height;
    return framebuffer;
}
```

**Replaces**: `vid_win.c` (line 36: `InitializeWindow()`), `vid_svgalib.c` (line 555: `VID_Init()`)

### 4.2 Headless Audio

**Enhanced**: `snd_null.c` (already exists with stub implementations)

Add audio capture capability for streaming:

```c
/* snd_null.c — Enhanced for streaming */
static int16_t *audio_capture_buffer;
static int capture_samples;

void SNDDMA_Submit(void) {
    /* Copy mixed audio to capture buffer for streaming */
    memcpy(audio_capture_buffer, paintbuffer, capture_samples * sizeof(int16_t));
}
```

### 4.3 Headless Input

**Enhanced**: `in_null.c` (already exists with stub implementations)

Add programmatic input injection for automated testing and remote control:

```c
/* in_null.c — Enhanced for remote input */
typedef struct remote_input_s {
    float forwardmove, sidemove, upmove;
    float viewangles[3];
    int buttons;
} remote_input_t;

static remote_input_t pending_input;

void IN_InjectInput(const remote_input_t *input) {
    pending_input = *input;
}
```

---

## 5. Container Architecture

### 5.1 Container Image Design

```dockerfile
# Multi-stage build
FROM ubuntu:22.04 AS builder
RUN apt-get update && apt-get install -y gcc cmake libsdl2-dev
COPY Quake/ /src/Quake/
WORKDIR /src/Quake
RUN cmake -B build -DHEADLESS=ON -DCMAKE_BUILD_TYPE=Release && \
    cmake --build build

FROM ubuntu:22.04 AS runtime
RUN useradd -m -s /bin/bash quake
COPY --from=builder /src/Quake/build/quake-server /usr/local/bin/
COPY --from=builder /src/Quake/build/quake-headless /usr/local/bin/
USER quake
EXPOSE 26000/udp
EXPOSE 8080/tcp
HEALTHCHECK --interval=10s CMD curl -f http://localhost:8080/healthz || exit 1
ENTRYPOINT ["quake-headless"]
```

### 5.2 Container Configuration

| Configuration | Source | Default |
|--------------|--------|---------|
| `QUAKE_PORT` | Environment variable | 26000 (from `net_main.c:34`) |
| `QUAKE_MAP` | Environment variable | `start` |
| `QUAKE_MAXPLAYERS` | Environment variable | 8 |
| `QUAKE_GAMEDIR` | Environment variable | `id1` |
| `QUAKE_MEMORY` | Environment variable | 16 (MB) |
| `QUAKE_HEADLESS` | Environment variable | `true` |
| `LOG_FORMAT` | Environment variable | `json` |
| `HEALTH_PORT` | Environment variable | 8080 |

### 5.3 Graceful Shutdown

```c
/* New: signal_handler.c */
#include <signal.h>

static volatile sig_atomic_t shutdown_requested = 0;

void handle_sigterm(int sig) {
    shutdown_requested = 1;
}

/* Integration point: host.c Host_Frame() */
void Host_Frame(float time) {
    if (shutdown_requested) {
        /* Drain connections, save state, then exit */
        Host_ShutdownServer(true);
        Sys_Quit();
    }
    /* ... existing frame logic ... */
}
```

---

## 6. Service Layer Architecture

### 6.1 Session Manager Service

Manages lifecycle of game engine instances:

```
┌──────────────────┐     ┌──────────────────┐
│  Session Manager │────▶│  Engine Container │
│  (Go/Python)     │     │  (Session 1)      │
│                  │────▶│  Engine Container │
│  REST API:       │     │  (Session 2)      │
│  POST /sessions  │────▶│  Engine Container │
│  GET  /sessions  │     │  (Session N)      │
│  DELETE /sessions│     │                    │
└──────────────────┘     └──────────────────┘
```

### 6.2 Streaming Gateway Service

Bridges game engine output to client browsers:

```
Client (Browser)  ←→  WebSocket/WebRTC  ←→  Gateway  ←→  Engine Container
                                              │
                                        Frame Buffer
                                        Video Encode
                                        Audio Capture
                                        Input Relay
```

---

## 7. Data Flow — Cloud Streaming

```
1. Client connects → Gateway (WebSocket)
2. Gateway requests session → Session Manager (REST)
3. Session Manager starts container → Kubernetes (Pod)
4. Engine runs headless → Renders frames to memory buffer
5. Gateway reads framebuffer → VID_GetFramebuffer()
6. Gateway encodes → H.264/VP8 video stream
7. Gateway captures audio → Audio capture buffer
8. Gateway streams to client → WebSocket/WebRTC
9. Client sends input → Gateway → Engine (IN_InjectInput)
10. Loop: Steps 4-9 at 30-60 FPS
```

---

## 8. Evolution Sequence

| Step | Change | Risk | Validation |
|------|--------|------|-----------|
| 1 | Define interface headers | None | Compiles |
| 2 | Implement headless video driver | Low | Engine starts without display |
| 3 | Implement headless audio/input | Low | Engine starts without hardware |
| 4 | Wire interfaces into engine initialization | Medium | Existing functionality preserved |
| 5 | Add framebuffer extraction API | Low | Can read rendered frames |
| 6 | Containerize headless engine | Medium | Container starts and runs |
| 7 | Build session manager | Medium | Multi-instance works |
| 8 | Build streaming gateway | High | End-to-end streaming works |
| 9 | Deploy to Azure | Medium | Cloud operational |
