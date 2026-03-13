# Architecture Review — Quake Engine

> Current architecture assessment for modernization planning.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

The Quake engine is a **monolithic, single-threaded C application** built in 1996 for DOS/Windows 95 and Linux/Solaris. The architecture tightly couples platform-specific code, rendering, networking, sound, and game logic into a single binary. While remarkably well-structured for its era, the engine lacks the modularity, abstraction layers, and service boundaries required for cloud-native deployment.

**Key Finding**: The codebase contains **240 `.c` files and 145 `.h` files** across WinQuake (128/68) and QuakeWorld (112/76), with **105+ global variables** governing engine state.

---

## 2. Current Architecture Pattern

### 2.1 Monolithic Single-Threaded Frame Loop

The engine follows a **single-threaded, frame-based simulation** pattern:

```
sys_win.c / sys_linux.c → Host_Init() → Host_Frame() loop → Host_Shutdown()
```

| Component | Entry Point | Source |
|-----------|-------------|--------|
| Main Loop | `Host_Frame()` | `legacy-src/desktop-engine/host.c:729` |
| Initialization | `Host_Init()` | `legacy-src/desktop-engine/host.c:835` |
| Shutdown | `Host_Shutdown()` | `legacy-src/desktop-engine/host.c:932` |
| Local Init | `Host_InitLocal()` | `legacy-src/desktop-engine/host.c:209` |
| Server Shutdown | `Host_ShutdownServer()` | `legacy-src/desktop-engine/host.c:405` |

Each frame processes input, runs server simulation (physics + QuakeC), runs client prediction, renders the scene, mixes audio, and sends/receives network packets — all sequentially in a single thread.

### 2.2 Component Coupling

```
┌─────────────────────────────────────────────────────────┐
│                    Platform Layer                         │
│  sys_win.c / sys_linux.c / sys_dos.c / sys_sun.c         │
│  vid_win.c / vid_svgalib.c / vid_x.c / vid_dos.c        │
│  in_win.c / in_dos.c / in_sun.c                          │
│  snd_win.c / snd_linux.c / snd_dos.c / snd_sun.c        │
│  cd_win.c / cd_linux.c / cd_null.c                       │
└────────────────────┬────────────────────────────────────┘
                     │ Direct function calls (no interfaces)
┌────────────────────▼────────────────────────────────────┐
│                    Engine Core                            │
│  host.c — Frame loop orchestrator                        │
│  common.c — File I/O, string ops, message parsing        │
│  cmd.c / cvar.c — Console command system                 │
│  zone.c — Custom memory allocator (Zone/Hunk/Cache)      │
│  model.c / gl_model.c — BSP/MDL/SPR loading              │
│  net_main.c / net_dgrm.c — Network protocol              │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│              Game Systems                                 │
│  sv_main.c / sv_phys.c / sv_move.c — Server simulation   │
│  cl_main.c / cl_parse.c / cl_input.c — Client logic      │
│  pr_exec.c / pr_edict.c / pr_cmds.c — QuakeC VM          │
│  r_main.c / gl_rmain.c — Rendering pipelines             │
│  snd_dma.c / snd_mix.c / snd_mem.c — Sound system        │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Tight Platform Coupling

**23 files** contain platform-specific `#ifdef` conditionals:

| Platform Guard | Files Affected | Examples |
|----------------|----------------|----------|
| `_WIN32` | 23 files | `host.c:908`, `snd_mix.c:51`, `menu.c`, `snd_dma.c:196` |
| `__linux__` | 5 files | `QW/client/cl_main.c`, `QW/client/sys_linux.c` |
| `GLQUAKE` | 12+ files | Renderer selection throughout |
| `id386` | 8+ files | Assembly optimization toggle |

Platform backends are selected at **compile time** via Makefile targets — there is no runtime abstraction or plugin system.

---

## 3. Rendering Pipeline Analysis

### 3.1 Dual Renderer Architecture

The engine ships two **compile-time exclusive** rendering paths:

| Renderer | Entry Point | Key Files |
|----------|-------------|-----------|
| Software | `R_RenderView()` at `r_main.c:1049` | `r_main.c`, `r_bsp.c`, `r_edge.c`, `d_scan.c`, `d_edge.c` |
| OpenGL | `R_RenderView()` at `gl_rmain.c:1104` | `gl_rmain.c`, `gl_rsurf.c`, `gl_mesh.c`, `gl_draw.c` |

Both renderers are tightly coupled to the engine — there is **no rendering interface or abstraction**. Switching requires recompilation with different source files.

### 3.2 Assembly Dependencies

**21 x86 assembly files** (10,748 lines in WinQuake) provide inner-loop optimizations:

| File | Lines | Purpose |
|------|-------|---------|
| `d_polysa.s` | 1,744 | Polygon rendering |
| `d_draw.s` | 1,037 | Pixel drawing |
| `d_draw16.s` | 974 | 16-bit drawing |
| `d_spr8.s` | 900 | Sprite rendering |
| `r_drawa.s` | 838 | General drawing |
| `surf8.s` | 783 | 8-bit surfaces |
| `r_edgea.s` | 750 | Edge processing |

These are **x86-only** (i386 AT&T syntax) and cannot run on ARM, RISC-V, or other architectures.

---

## 4. Memory Management Architecture

The engine implements a **three-tiered custom memory allocator** in `legacy-src/desktop-engine/zone.c`:

| Allocator | Function | Line | Use Case |
|-----------|----------|------|----------|
| **Zone** | `Z_Malloc()` | `zone.c:142` | Small, temporary allocations (strings, commands) |
| **Hunk** | `Hunk_AllocName()` | `zone.c:399` | Large, named blocks (models, maps, textures) |
| **Cache** | `Cache_Alloc()` | `zone.c:871` | Swappable data (sounds, model frames) |

The entire engine operates within a **single pre-allocated memory block** (typically 8-16 MB), passed via `host_parms.membase` at startup (`host.c:835`). This design:

- Prevents use of standard `malloc`/`free` for most allocations
- Makes memory profiling and debugging difficult
- Causes hard failures when memory is exhausted (no graceful degradation)
- Is incompatible with modern container memory management

---

## 5. Network Architecture

### 5.1 WinQuake (LAN-Focused)

| Component | Source | Architecture |
|-----------|--------|-------------|
| Protocol | `net_dgrm.c` | Reliable + unreliable channels, sequence numbering |
| Transport | `net_udp.c`, `net_wins.c`, `net_wipx.c` | Driver-based (UDP, IPX, serial) |
| Discovery | `net_main.c` | LAN broadcast for server discovery |

**Hard-coded port**: `net_main.c:34` — `DEFAULTnet_hostport = 26000`

### 5.2 QuakeWorld (Internet-Optimized)

| Component | Source | Architecture |
|-----------|--------|-------------|
| Delta Compression | `QW/server/sv_ents.c:155` | Bitmask-based field differencing |
| Client Prediction | `QW/client/cl_pred.c` | Local physics simulation |
| Entity Parsing | `QW/client/cl_ents.c:265` | `CL_ParsePacketEntities()` |

---

## 6. Key Architecture Gaps for Cloud-Native

| Gap | Current State | Cloud-Native Requirement |
|-----|---------------|-------------------------|
| **No service boundaries** | Single monolithic binary | Microservices with APIs |
| **No configuration management** | Hard-coded values, cvars | Environment variables, config maps |
| **No health checks** | Process lives or dies | HTTP health/readiness probes |
| **No observability** | `printf`-based debug output | Structured logging, metrics, tracing |
| **No graceful shutdown** | `Sys_Quit()` / `exit()` | SIGTERM handling, connection draining |
| **No state persistence** | In-memory only | External state store |
| **No horizontal scaling** | Single server instance | Multi-instance orchestration |
| **Platform lock-in** | Compile-time platform selection | Runtime platform abstraction |
| **x86 dependency** | Assembly inner loops | Portable C or SIMD intrinsics |
| **Custom memory** | Zone/Hunk/Cache allocator | Standard allocator + container limits |

---

## 7. Modernization Impact Assessment

### 7.1 Files Requiring Major Changes

| Category | File Count | Examples |
|----------|-----------|----------|
| Platform abstraction | 20+ | `sys_*.c`, `vid_*.c`, `in_*.c`, `snd_*.c`, `cd_*.c` |
| Renderer modularization | 15+ | `r_*.c`, `gl_*.c`, `d_*.c` |
| Network modernization | 12+ | `net_*.c`, `QW/client/cl_ents.c`, `QW/server/sv_ents.c` |
| Memory management | 1 core + 50+ callers | `zone.c` and all `Hunk_Alloc`/`Z_Malloc` callers |
| Build system | 6 build files | All Makefiles and `.dsp`/`.dsw` files |

### 7.2 Estimated Complexity

| Modernization Phase | Estimated Effort | Risk Level |
|---------------------|-----------------|------------|
| Build system migration (CMake) | 2-3 weeks | Low |
| Security remediation | 3-4 weeks | Medium |
| Platform abstraction layer | 4-6 weeks | High |
| Renderer modularization | 6-8 weeks | High |
| Containerization | 2-3 weeks | Medium |
| Cloud deployment (Azure) | 4-6 weeks | Medium |
| Streaming pipeline | 8-12 weeks | Very High |

---

## 8. Recommendations

1. **Start with build system** — CMake migration enables all subsequent work
2. **Security remediation early** — Buffer overflow fixes are prerequisite for any network-facing deployment
3. **Headless mode before containerization** — Extract rendering to enable server-only containers
4. **Incremental approach** — Keep the engine functional at every step; maintain a working build throughout
5. **Assembly as last migration** — Replace `.s` files with portable C only after all other refactoring is stable
