# Performance Analysis — Quake Engine

> Performance bottleneck assessment for modernization planning.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

The Quake engine was highly optimized for 1996-era single-core x86 processors with limited memory. Its performance characteristics are dominated by **CPU-bound software rendering**, **single-threaded execution**, **custom memory management**, and **x86 assembly inner loops**. For cloud-native deployment with game streaming, the key bottlenecks shift to **frame buffer extraction**, **video encoding latency**, **network protocol overhead**, and **concurrent session scaling**.

---

## 2. Rendering Performance

### 2.1 Software Renderer Bottlenecks

The software renderer is entirely CPU-bound with critical inner loops:

| Bottleneck | File | Line | Description |
|------------|------|------|-------------|
| **Span rasterization** | `legacy-src/desktop-engine/d_scan.c` | 274-382 | `D_DrawSpans8()` — Main texture-mapped span loop. Pixel-by-pixel iteration at line 370-375 |
| **Turbulent water** | `legacy-src/desktop-engine/d_scan.c` | 100-112 | `D_DrawTurbulent8Span()` — Per-pixel warping for water surfaces |
| **Z-buffer spans** | `legacy-src/desktop-engine/d_scan.c` | 395+ | `D_DrawZSpans()` — Z-buffer fill |
| **Edge processing** | `legacy-src/desktop-engine/d_edge.c` | 123 | `D_CalcGradients()` — Surface gradient computation per surface |
| **Surface dispatch** | `legacy-src/desktop-engine/d_edge.c` | 174-204 | `D_DrawSurfaces()` — Surface type dispatch with iteration |
| **BSP traversal** | `legacy-src/desktop-engine/r_bsp.c` | varies | `R_RecursiveWorldNode()` — Recursive front-to-back BSP traversal |
| **Edge sorting** | `legacy-src/desktop-engine/r_edge.c` | varies | Edge-based scan conversion and sorting |

**Assembly optimized inner loops** (21 files, 10,748 lines in WinQuake):

| File | Lines | Critical Path |
|------|-------|--------------|
| `d_polysa.s` | 1,744 | Polygon setup and rasterization |
| `d_draw.s` | 1,037 | Core pixel drawing loops |
| `d_draw16.s` | 974 | 16-bit pixel drawing |
| `d_spr8.s` | 900 | Sprite rasterization |
| `r_drawa.s` | 838 | General drawing operations |
| `surf8.s` | 783 | 8-bit surface blitting |
| `r_edgea.s` | 750 | Edge processing |
| `d_parta.s` | 477 | Particle rendering |
| `math.s` | 418 | Vector/matrix math |
| `snd_mixa.s` | 218 | Sound mixing |
| `r_aliasa.s` | 237 | Alias model rendering |

**Performance Impact**: These assembly routines provide 2-5x speedup over equivalent C code on x86. Removing them without replacement will cause significant performance regression.

### 2.2 OpenGL Renderer

| Component | File | Line | Performance Note |
|-----------|------|------|-----------------|
| Entry point | `legacy-src/desktop-engine/gl_rmain.c` | 1104 | `R_RenderView()` — GPU-bound on modern hardware |
| Surface rendering | `legacy-src/desktop-engine/gl_rsurf.c` | varies | Lightmap updates are CPU-bound |
| Mesh rendering | `legacy-src/desktop-engine/gl_mesh.c` | varies | Per-vertex transformation (OpenGL 1.1) |
| Texture management | `legacy-src/desktop-engine/gl_draw.c` | varies | Immediate mode GL calls (no VBO/VAO) |

**Key Issue**: Uses **OpenGL 1.1 immediate mode** — no vertex buffers, no shaders, no modern GPU pipeline utilization.

---

## 3. Network Performance

### 3.1 WinQuake Protocol Overhead

| Bottleneck | File | Line | Description |
|------------|------|------|-------------|
| **Packet transmission** | `legacy-src/desktop-engine/net_dgrm.c` | 161 | `Datagram_SendMessage()` — Reliable message with ACK/retransmit |
| **Message fragmentation** | `legacy-src/desktop-engine/net_dgrm.c` | varies | MAX_DATAGRAM = 1024 bytes forces fragmentation |
| **Full state broadcasts** | `legacy-src/desktop-engine/sv_main.c` | varies | Entire entity state sent each frame |

### 3.2 QuakeWorld Protocol (Improved)

| Component | File | Line | Performance |
|-----------|------|------|-------------|
| **Delta encoding** | `legacy-src/QW/server/sv_ents.c` | 155 | `SV_WriteDelta()` — Bitmask field differencing |
| **Packet building** | `legacy-src/QW/server/sv_ents.c` | 250 | `SV_EmitPacketEntities()` — Delta-compressed entity updates |
| **Entity parsing** | `legacy-src/QW/client/cl_ents.c` | 265 | `CL_ParsePacketEntities()` — Delta decompression |
| **Entity linking** | `legacy-src/QW/client/cl_ents.c` | 408 | `CL_LinkPacketEntities()` — Scene integration |

**Optimization Opportunity**: QuakeWorld's delta compression is already efficient but operates over raw UDP without congestion control, MTU discovery, or quality-of-service guarantees.

---

## 4. Memory Performance

### 4.1 Zone Allocator

| Function | File | Line | Bottleneck |
|----------|------|------|-----------|
| `Z_Malloc()` | `legacy-src/desktop-engine/zone.c` | 142 | Wrapper — calls Z_TagMalloc |
| `Z_TagMalloc()` | `legacy-src/desktop-engine/zone.c` | 155 | **Linear free-list search** at line 182 — O(n) allocation |
| `Z_Free()` | `legacy-src/desktop-engine/zone.c` | 99 | Block merging with validation |
| `Z_CheckHeap()` | `legacy-src/desktop-engine/zone.c` | 247 | Full heap walk — O(n) debug validation |

### 4.2 Hunk Allocator

| Function | File | Line | Characteristic |
|----------|------|------|---------------|
| `Hunk_AllocName()` | `legacy-src/desktop-engine/zone.c` | 399 | Stack-based — O(1) allocation, no deallocation |
| `Hunk_Alloc()` | `legacy-src/desktop-engine/zone.c` | 434 | Unnamed wrapper |

**Heavy callers**:
- `legacy-src/desktop-engine/gl_model.c` — 25+ `Hunk_AllocName` calls for model/texture loading
- `legacy-src/desktop-engine/model.c` — 25+ calls for BSP geometry loading
- `legacy-src/desktop-engine/host.c` — Lines 195, 915 for engine initialization

### 4.3 Cache Allocator

| Function | File | Line | Characteristic |
|----------|------|------|---------------|
| `Cache_Alloc()` | `legacy-src/desktop-engine/zone.c` | 871 | LRU eviction — can cause stalls on cache miss |

**Performance Issue**: Cache eviction during gameplay causes frame stutters when sounds or model data must be reloaded.

### 4.4 Fixed Memory Limits

| Resource | Limit | Impact |
|----------|-------|--------|
| Zone pool | 49 KB (`zone.c:24`) | Extremely limited dynamic allocation |
| Total memory | 8-16 MB typical | Entire engine + game data must fit |
| Entity slots | 1024 (`bspfile.h:28`) | Limits game complexity |
| Visible entities | 256 (`client.h:306`) | Limits visible scene complexity |

---

## 5. Sound Performance

### 5.1 Sound Mixing Pipeline

| Function | File | Line | Bottleneck |
|----------|------|------|-----------|
| **Main mixing loop** | `legacy-src/desktop-engine/snd_mix.c` | 261 | `S_PaintChannels()` — Outer time loop at line 269 |
| **Channel iteration** | `legacy-src/desktop-engine/snd_mix.c` | 281 | `for (i=0; i<total_channels; i++, ch++)` |
| **Sample processing** | `legacy-src/desktop-engine/snd_mix.c` | 293 | Inner `while (ltime < end)` loop |
| **8-bit mixing** | `legacy-src/desktop-engine/snd_mix.c` | 346-362 | `SND_PaintChannelFrom8()` — Per-sample loop |
| **16-bit mixing** | `legacy-src/desktop-engine/snd_mix.c` | 375-387 | `SND_PaintChannelFrom16()` — Per-sample loop |
| **Buffer transfer** | `legacy-src/desktop-engine/snd_mix.c` | 65-87 | `S_TransferStereo16()` — Retry loop for buffer locking |
| **Assembly mixing** | `legacy-src/desktop-engine/snd_mixa.s` | 218 lines | x86-optimized mixing inner loops |

**Supported sample rates**: 8000, 11025, 16000, 22051, 32000, 44100, 48000 Hz (`snd_gus.c:563-574`, `snd_linux.c:35`)

---

## 6. Cloud Streaming Performance Considerations

### 6.1 Frame Buffer Extraction

**Current State**: The engine writes rendered frames to:
- Software: Direct pixel buffer (`vid.buffer` pointer)
- OpenGL: GPU framebuffer via OpenGL 1.1

**For streaming**: Frame data must be extracted at 30-60 FPS with minimal latency:
- Software renderer: Direct memory read (fast, ~1ms for 640×480)
- OpenGL renderer: `glReadPixels()` (slow, GPU→CPU transfer stall)

### 6.2 Video Encoding Pipeline (Not Yet Implemented)

Required encoding path for cloud streaming:

```
Frame Buffer → Color Space Conversion → Video Encode → Network Transmit
  ~1ms            ~1ms (SIMD)            ~5-16ms (H.264)    ~1-5ms
```

**Target latency budget**: < 50ms end-to-end for playable streaming

### 6.3 Concurrent Session Scaling

| Resource | Per Session | 10 Sessions | 100 Sessions |
|----------|-------------|-------------|--------------|
| CPU (software render) | 1 core | 10 cores | 100 cores |
| Memory | 16-64 MB | 160-640 MB | 1.6-6.4 GB |
| Network (delta) | 20-50 Kbps | 200-500 Kbps | 2-5 Mbps |
| Video stream out | 2-5 Mbps | 20-50 Mbps | 200-500 Mbps |

**Bottleneck**: Video encoding at scale — each session needs a dedicated encoder thread or hardware encoder.

---

## 7. Performance Optimization Roadmap

| Phase | Optimization | Expected Impact | Effort |
|-------|-------------|-----------------|--------|
| 1 | Replace assembly with SIMD intrinsics | Maintain parity, enable multi-arch | High |
| 2 | Multi-thread rendering (separate render thread) | 1.5-2x frame rate | High |
| 3 | Modern OpenGL (VBO/VAO/shaders) | 10-100x GPU throughput | Medium |
| 4 | Hardware video encoding (VAAPI/NVENC) | 10x encode speed | Medium |
| 5 | Frame buffer double-buffering for streaming | Eliminate frame extraction stalls | Low |
| 6 | Connection pooling and multiplexing | Reduce per-session overhead | Medium |
| 7 | Memory pool per session | Enable container density | Medium |
