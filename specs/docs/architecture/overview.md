# Architecture Overview — Quake Engine

> Reverse-engineered from the id Software Quake source code (1996-1997).
> All references point to actual source files in this repository.

---

## 1. High-Level Architecture

The Quake engine follows a **single-threaded, frame-based simulation** architecture with a clear separation between the platform abstraction layer and the engine core.

```mermaid
graph TB
    subgraph "Application Layer"
        HL[host.c — Main Loop]
    end

    subgraph "Game Systems"
        SV[Server<br/>sv_main.c, sv_phys.c]
        CL[Client<br/>cl_main.c, cl_parse.c]
        QC[QuakeC VM<br/>pr_exec.c, pr_edict.c]
    end

    subgraph "Engine Services"
        NET[Network<br/>net_main.c, net_dgrm.c]
        SND[Sound<br/>snd_dma.c, snd_mix.c]
        CMD[Console & Commands<br/>cmd.c, cvar.c, console.c]
        MDL[Model/BSP Loader<br/>model.c, bspfile.h]
    end

    subgraph "Rendering"
        SW[Software Renderer<br/>r_main.c, d_edge.c]
        GL[OpenGL Renderer<br/>gl_rmain.c, gl_rsurf.c]
    end

    subgraph "Platform Abstraction"
        SYS[System<br/>sys_win.c, sys_linux.c]
        VID[Video<br/>vid_win.c, vid_svgalib.c]
        INP[Input<br/>in_win.c, keys.c]
        CDA[CD Audio<br/>cd_win.c, cd_linux.c]
        SNDB[Sound Backend<br/>snd_win.c, snd_linux.c]
    end

    HL --> SV
    HL --> CL
    HL --> CMD
    SV --> QC
    SV --> NET
    CL --> NET
    CL --> SW
    CL --> GL
    CL --> SND
    SV --> MDL
    CL --> MDL
    SW --> VID
    GL --> VID
    INP --> CL
    SND --> SNDB
    SYS --> HL
```

---

## 2. Component Overview

### 2.1 Main Loop (`Quake/WinQuake/host.c`)

The engine runs a single-threaded main loop via `Host_Frame()` (line ~729):

```
Host_Frame()
├── Sys_SendKeyEvents()        — Pump OS input events
├── Cbuf_Execute()             — Execute queued commands
├── Host_ServerFrame()         — Update server (if hosting)
│   ├── SV_Physics()           — Run physics for all entities
│   └── SV_SendClientMessages()— Send state to clients
├── Host_ClientFrame()         — Update client (if connected)
│   ├── CL_ReadPackets()       — Parse server messages
│   ├── CL_SendCmd()           — Send movement to server
│   └── CL_SetUpPlayerPrediction() — Predict local player
├── SCR_UpdateScreen()         — Render frame
│   └── R_RenderView()         — 3D scene rendering
└── S_Update()                 — Update audio spatialization
```

### 2.2 Two Engine Variants

| Variant | Path | Purpose |
|---------|------|---------|
| **WinQuake** | `Quake/WinQuake/` | Full engine for single-player + LAN multiplayer |
| **QuakeWorld** | `Quake/QW/client/` + `Quake/QW/server/` | Internet-optimized multiplayer with separated client & dedicated server |

Key architectural differences:

| Feature | WinQuake | QuakeWorld |
|---------|----------|------------|
| Networking | Reliable datagram, full entity state | Delta compression, unreliable streams |
| Prediction | None (server-authoritative only) | Client-side prediction (`cl_pred.c`) |
| Server | Integrated with client | Dedicated server binary (`qwsv`) |
| Spectators | Not supported | Full spectator mode |
| Protocol | Simple entity updates | Flag-based delta encoding |

---

## 3. Data Flow

### 3.1 Single-Player Data Flow (WinQuake)

```mermaid
sequenceDiagram
    participant Input as Input System
    participant Client as Client
    participant Server as Server
    participant QC as QuakeC VM
    participant Renderer as Renderer
    participant Sound as Sound

    Input->>Client: Key/mouse events
    Client->>Server: usercmd_t (movement)
    Server->>QC: PlayerPreThink / PlayerPostThink
    QC->>Server: Entity updates
    Server->>Server: SV_Physics()
    Server->>Client: Entity state snapshot
    Client->>Renderer: R_RenderView()
    Client->>Sound: S_Update() (3D audio)
```

### 3.2 QuakeWorld Multiplayer Data Flow

```mermaid
sequenceDiagram
    participant Input as Input System
    participant Client as QW Client
    participant Network as UDP Network
    participant Server as QW Server
    participant QC as QuakeC VM

    Input->>Client: Key/mouse events
    Client->>Client: CL_PredictMove() (local prediction)
    Client->>Network: clc_move (usercmd_t)
    Network->>Server: Receive user commands
    Server->>QC: Execute game logic
    Server->>Server: SV_Physics()
    Server->>Network: svc_packetentities (delta-compressed)
    Network->>Client: Receive delta updates
    Client->>Client: CL_ParseDelta() (decompress)
    Client->>Client: Reconcile prediction with server state
```

---

## 4. Memory Architecture

The engine uses a custom memory management system (`Quake/WinQuake/zone.c`):

```mermaid
graph LR
    subgraph "Memory Zones"
        HUNK[Hunk Memory<br/>Long-lived: models, BSP, textures]
        ZONE[Zone Memory<br/>Medium-lived: buffers, strings]
        CACHE[Cache Memory<br/>Temporary: can be evicted]
        TEMP[Temp Memory<br/>Per-frame allocations]
    end

    HUNK -->|"Mod_LoadBrushModel()"| BSP[BSP Data]
    HUNK -->|"Mod_LoadAliasModel()"| MDL[Model Data]
    ZONE -->|"Z_Malloc()"| BUF[Network Buffers]
    CACHE -->|"Cache_Alloc()"| SFX[Sound Effects]
    TEMP -->|"Hunk_TempAlloc()"| FRAME[Frame Data]
```

| Zone | Lifetime | Usage | Source |
|------|----------|-------|--------|
| **Hunk** | Level lifetime | BSP data, models, textures | `Quake/WinQuake/zone.c` |
| **Zone** | Variable | Network buffers, command strings | `Quake/WinQuake/zone.c` |
| **Cache** | Evictable | Sound samples, model skins | `Quake/WinQuake/zone.c` |
| **Temp** | Per-frame | Temporary calculations | `Quake/WinQuake/zone.c` |

---

## 5. Build Targets

### WinQuake Targets

| Target | Platform | Renderer | Build File |
|--------|----------|----------|------------|
| `squake` | Linux (SVGAlib) | Software | `Quake/WinQuake/Makefile.linuxi386` |
| `glquake` | Linux (Mesa) | OpenGL | `Quake/WinQuake/Makefile.linuxi386` |
| `glquake.glx` | Linux (X11 GLX) | OpenGL | `Quake/WinQuake/Makefile.linuxi386` |
| `quake.x11` | Linux (X11) | Software | `Quake/WinQuake/Makefile.linuxi386` |
| `WinQuake.exe` | Windows | Software + GL | `Quake/WinQuake/WinQuake.dsp` |
| `quake.sw` | Solaris (X11) | Software | `Quake/WinQuake/Makefile.Solaris` |

### QuakeWorld Targets

| Target | Platform | Description | Build File |
|--------|----------|-------------|------------|
| `qwsv` | Linux/Solaris | Dedicated server | `Quake/QW/Makefile.Linux` |
| `qwcl` | Linux (SVGAlib) | Software client | `Quake/QW/Makefile.Linux` |
| `qwcl.x11` | Linux (X11) | Software client | `Quake/QW/Makefile.Linux` |
| `glqwcl` | Linux (Mesa) | OpenGL client | `Quake/QW/Makefile.Linux` |
| `glqwcl.glx` | Linux (X11 GLX) | OpenGL client | `Quake/QW/Makefile.Linux` |

### Utility Targets

| Target | Description | Build File |
|--------|-------------|------------|
| `gas2masm` | GAS → MASM assembly converter | `Quake/QW/gas2masm/gas2masm.dsp` |
| `qwfwd` | QuakeWorld UDP forwarding proxy | `Quake/QW/qwfwd/` |
| `qwprogs.dat` | Compiled QuakeC game logic | `Quake/qw-qc/progs.src` |

---

## 6. Key Architectural Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **Single-threaded** | Deterministic behavior, simpler debugging | Cannot leverage multiple CPU cores |
| **BSP tree rendering** | Efficient visibility determination for indoor scenes | Poor for large outdoor areas |
| **Server-authoritative** | Prevents cheating, consistent game state | Adds latency for client actions |
| **QuakeC scripting** | Rapid game logic iteration without recompilation | Performance overhead vs native code |
| **Custom memory management** | Predictable allocation patterns, no fragmentation | Complexity, fixed pool sizes |
| **Platform abstraction via `sys_*.c`** | Easy porting to new platforms | Code duplication across platform files |
| **Delta compression (QW)** | Massive bandwidth reduction for internet play | Implementation complexity, debugging difficulty |
| **Client-side prediction (QW)** | Responsive gameplay despite network latency | Prediction errors cause visible corrections |
