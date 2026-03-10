# Component Architecture — Quake Engine

> Reverse-engineered from the id Software Quake source code (1996-1997).
> All file references point to actual source files in this repository.

---

## 1. Rendering Subsystem

### 1.1 Software Renderer

The software renderer implements a **span-based rasterizer** with **edge sorting** for correct visibility.

```mermaid
graph TD
    RM[r_main.c<br/>R_RenderView] --> RB[r_bsp.c<br/>R_RenderWorld]
    RB --> RE[r_edge.c<br/>Edge sorting & scan conversion]
    RE --> RS[r_surf.c<br/>Surface rendering]
    RS --> DS[d_scan.c<br/>Scan-line rasterization]
    DS --> DE[d_edge.c<br/>Edge processing]
    DE --> DF[d_surf.c<br/>Surface cache management]

    RM --> RA[r_alias.c<br/>Alias model rendering]
    RM --> RP[r_sprite.c<br/>Sprite rendering]
    RM --> RK[r_sky.c<br/>Sky rendering]
    RM --> RL[r_light.c<br/>Dynamic lighting]
    RM --> RPT[r_part.c<br/>Particle system]

    DE --> DI[d_init.c<br/>Rasterizer initialization]
    DS --> DV[d_vars.c<br/>Rasterizer globals]
```

**Key files and responsibilities:**

| File | Path | Purpose | Key Functions |
|------|------|---------|---------------|
| `r_main.c` | `Quake/WinQuake/` | Rendering entry point, frustum setup | `R_RenderView()`, `R_SetupFrame()` |
| `r_bsp.c` | `Quake/WinQuake/` | BSP tree traversal, visible surface determination | `R_RenderWorld()`, `R_RecursiveWorldNode()` |
| `r_edge.c` | `Quake/WinQuake/` | Edge-based scan conversion | `R_EdgeDrawing()`, `R_ScanEdges()` |
| `r_surf.c` | `Quake/WinQuake/` | Surface rendering with lightmaps | `R_DrawSurface()`, `R_BuildLightMap()` |
| `r_alias.c` | `Quake/WinQuake/` | Animated mesh model rendering | `R_AliasDrawModel()` |
| `r_light.c` | `Quake/WinQuake/` | Dynamic light contribution | `R_AddDynamicLights()`, `R_MarkLights()` |
| `r_part.c` | `Quake/WinQuake/` | Particle effects | `R_DrawParticles()` |
| `r_sky.c` | `Quake/WinQuake/` | Scrolling sky texture | `R_DrawSkyChain()` |
| `d_edge.c` | `Quake/WinQuake/` | Low-level edge processing | `D_DrawSurfaces()` |
| `d_scan.c` | `Quake/WinQuake/` | Affine texture mapping spans | `D_DrawSpans8()` |
| `d_surf.c` | `Quake/WinQuake/` | Surface cache management | `D_CacheSurface()` |

**Key data structures (`Quake/WinQuake/model.h`, `Quake/WinQuake/r_local.h`):**

| Structure | Purpose |
|-----------|---------|
| `mnode_t` | BSP tree node (splitting plane + children) |
| `mleaf_t` | BSP leaf (contains PVS data, surfaces, entities) |
| `msurface_t` | Renderable surface (texture, lightmap, edges) |
| `mplane_t` | Splitting plane (normal vector + distance + type) |
| `surfcache_t` | Cached rendered surface for reuse |

### 1.2 OpenGL Renderer

The OpenGL renderer replaces the software rasterizer with hardware-accelerated polygon rendering.

| File | Path | Purpose | Key Functions |
|------|------|---------|---------------|
| `gl_rmain.c` | `Quake/WinQuake/` | GL rendering entry point | `R_RenderView()`, `R_DrawEntitiesOnList()` |
| `gl_rsurf.c` | `Quake/WinQuake/` | GL surface/BSP rendering | `R_DrawBrushModel()`, `R_DrawWorld()` |
| `gl_mesh.c` | `Quake/WinQuake/` | GL alias model mesh rendering | `GL_MakeAliasModelDisplayLists()` |
| `gl_rlight.c` | `Quake/WinQuake/` | GL dynamic lighting | `R_RenderDlights()` |
| `gl_draw.c` | `Quake/WinQuake/` | GL 2D drawing (HUD, console) | `GL_Upload8()`, `Draw_Pic()` |
| `gl_warp.c` | `Quake/WinQuake/` | GL sky and water warping | `EmitWaterPolys()`, `R_DrawSkyChain()` |
| `gl_model.c` | `Quake/WinQuake/` | GL-specific model loading | `Mod_LoadBrushModel()` |
| `gl_refrag.c` | `Quake/WinQuake/` | GL entity fragment linking | `R_StoreEfrags()` |
| `gl_screen.c` | `Quake/WinQuake/` | GL screen management | `SCR_UpdateScreen()` |
| `gl_vidnt.c` | `Quake/WinQuake/` | GL Windows video driver | `VID_Init()` |
| `gl_vidlinux.c` | `Quake/WinQuake/` | GL Linux SVGA video driver | `VID_Init()` |
| `gl_vidlinuxglx.c` | `Quake/WinQuake/` | GL Linux X11/GLX video driver | `VID_Init()` |

**Defined by:** `GLQUAKE` preprocessor macro

---

## 2. Networking Subsystem

### 2.1 WinQuake Networking

```mermaid
graph TD
    NM[net_main.c<br/>Network Manager] --> NL[net_loop.c<br/>Loopback Driver]
    NM --> ND[net_dgrm.c<br/>Datagram Protocol]
    ND --> NU[net_udp.c<br/>UDP/BSD Sockets]
    ND --> NW[net_wins.c<br/>Windows Sockets]
    ND --> NI[net_wipx.c<br/>Windows IPX]
    ND --> NS[net_ser.c<br/>Serial/Modem]
    ND --> NX[net_ipx.c<br/>DOS IPX]
```

| File | Path | Purpose |
|------|------|---------|
| `net_main.c` | `Quake/WinQuake/` | Network initialization, message dispatch, driver management |
| `net_dgrm.c` | `Quake/WinQuake/` | Reliable datagram protocol (sequence numbers, ACK/retransmit) |
| `net_loop.c` | `Quake/WinQuake/` | Loopback driver for single-machine play |
| `net_udp.c` | `Quake/WinQuake/` | BSD UDP socket implementation |
| `net_wins.c` | `Quake/WinQuake/` | Windows Winsock UDP |
| `net_wipx.c` | `Quake/WinQuake/` | Windows IPX/SPX protocol |
| `net_ser.c` | `Quake/WinQuake/` | Serial/modem point-to-point |

**Key data structure — `qsocket_t` (`Quake/WinQuake/net.h`):**
- Connection state: `connecttime`, `lastMessageTime`, `disconnected`
- Reliable messaging: `ackSequence`, `sendSequence`, `sendMessage[]`
- Receive tracking: `receiveSequence`, `receiveMessage[]`

### 2.2 QuakeWorld Networking

QuakeWorld replaces the WinQuake networking with an internet-optimized system:

| File | Path | Purpose |
|------|------|---------|
| `net_chan.c` | `Quake/QW/client/` | Network channel with reliable + unreliable streams |
| `cl_ents.c` | `Quake/QW/client/` | Entity delta decompression |
| `cl_pred.c` | `Quake/QW/client/` | Client-side movement prediction |
| `sv_ents.c` | `Quake/QW/server/` | Entity delta compression for server |
| `sv_send.c` | `Quake/QW/server/` | Server message sending, PVS culling |
| `sv_nchan.c` | `Quake/QW/server/` | Server reliable channel management |

**Delta compression protocol (`Quake/QW/client/protocol.h`):**

| Flag | Bit | Field |
|------|-----|-------|
| `U_ORIGIN1` | `1<<9` | X origin (1/8 unit precision) |
| `U_ORIGIN2` | `1<<10` | Y origin |
| `U_ORIGIN3` | `1<<11` | Z origin |
| `U_ANGLE2` | `1<<12` | Pitch angle (byte) |
| `U_FRAME` | `1<<13` | Animation frame |
| `U_REMOVE` | `1<<14` | Entity removed |
| `U_MOREBITS` | `1<<15` | Extended flags follow |

---

## 3. QuakeC Virtual Machine

```mermaid
graph TD
    PROGS[progs.dat / qwprogs.dat<br/>Compiled bytecode] --> EXEC[pr_exec.c<br/>Bytecode interpreter]
    EXEC --> EDICT[pr_edict.c<br/>Entity management]
    EXEC --> CMDS[pr_cmds.c<br/>~100 builtin functions]
    EDICT --> WORLD[world.c<br/>Spatial queries]
    CMDS --> SV[Server subsystem]
    CMDS --> SND[Sound subsystem]
    CMDS --> NET[Network messaging]
```

| File | Path | Purpose | Key Functions |
|------|------|---------|---------------|
| `pr_exec.c` | `Quake/WinQuake/` | Bytecode execution loop | `PR_ExecuteProgram()`, `PR_EnterFunction()` |
| `pr_edict.c` | `Quake/WinQuake/` | Entity allocation, field access, progs loading | `ED_Alloc()`, `ED_Free()`, `PR_LoadProgs()` |
| `pr_cmds.c` | `Quake/WinQuake/` | Engine-to-script bridge (builtins) | `PF_traceline()`, `PF_sound()`, `PF_spawn()` |
| `pr_comp.h` | `Quake/WinQuake/` | Opcode definitions, instruction format | ~80 opcodes defined |
| `progs.h` | `Quake/WinQuake/` | VM data structures | `edict_t`, `dstatement_t`, `dfunction_t` |

**Instruction format (`Quake/WinQuake/pr_comp.h`):**
```
typedef struct {
    unsigned short op;      // Opcode
    unsigned short a, b, c; // Operand indices into globals
} dstatement_t;
```

**Opcode categories:**
- Arithmetic: `OP_MUL_F`, `OP_DIV_F`, `OP_ADD_F`, `OP_SUB_F`, `OP_ADD_V`, `OP_SUB_V`
- Comparison: `OP_EQ_F`, `OP_NE_F`, `OP_LE`, `OP_GE`, `OP_LT`, `OP_GT`
- Load/Store: `OP_LOAD_F`, `OP_STORE_F`, `OP_STOREP_F`
- Control flow: `OP_IF`, `OP_IFNOT`, `OP_GOTO`, `OP_CALL0`–`OP_CALL8`, `OP_RETURN`, `OP_DONE`

---

## 4. Sound Subsystem

```mermaid
graph TD
    SDMA[snd_dma.c<br/>Sound Manager] --> SMIX[snd_mix.c<br/>Channel Mixing]
    SDMA --> SMEM[snd_mem.c<br/>Sample Loading]
    SDMA --> SPAT[Spatialization<br/>3D panning + attenuation]

    SMIX --> SW[snd_win.c<br/>Windows DirectSound]
    SMIX --> SL[snd_linux.c<br/>Linux OSS /dev/dsp]
    SMIX --> SS[snd_sun.c<br/>Solaris audio]
    SMIX --> SD[snd_dos.c<br/>DOS DMA]
    SMIX --> SG[snd_gus.c<br/>Gravis UltraSound]
    SMIX --> SN[snd_null.c<br/>Null driver]

    CD[cd_audio.c<br/>CD Music] --> CDW[cd_win.c]
    CD --> CDL[cd_linux.c]
    CD --> CDN[cd_null.c]
```

| File | Path | Purpose |
|------|------|---------|
| `snd_dma.c` | `Quake/WinQuake/` | Main sound system: channel management, spatialization, DMA control |
| `snd_mix.c` | `Quake/WinQuake/` | Audio mixing of multiple channels into DMA buffer |
| `snd_mem.c` | `Quake/WinQuake/` | WAV file loading and sample memory management |
| `sound.h` | `Quake/WinQuake/` | Sound structures: `channel_t`, `sfx_t`, `dma_t` |

**Key data structures (`Quake/WinQuake/sound.h`):**

| Structure | Purpose |
|-----------|---------|
| `dma_t` | DMA buffer descriptor (channels, sample rate, bit depth, buffer pointer) |
| `channel_t` | Playing sound channel (position, volume, attenuation, entity) |
| `sfx_t` | Sound effect handle (name, cache pointer) |
| `sfxcache_t` | Cached sound sample data |

---

## 5. Input Subsystem

| File | Path | Purpose |
|------|------|---------|
| `in_win.c` | `Quake/WinQuake/` | Windows input: mouse (DirectInput), joystick (6-axis), keyboard |
| `in_dos.c` | `Quake/WinQuake/` | DOS input: mouse, keyboard via interrupts |
| `in_sun.c` | `Quake/WinQuake/` | Solaris input via X11 events |
| `in_null.c` | `Quake/WinQuake/` | Null input driver (stub) |
| `keys.c` | `Quake/WinQuake/` | Key binding management, key event dispatch |
| `keys.h` | `Quake/WinQuake/` | Key code definitions |
| `input.h` | `Quake/WinQuake/` | Input interface: `IN_Init()`, `IN_Move()`, `IN_Commands()` |

**User command structure (`Quake/WinQuake/protocol.h`):**
```
typedef struct {
    byte    msec;           // Milliseconds since last command
    byte    buttons;        // Button bitmask
    short   forwardmove;    // -127 to 127
    short   sidemove;       // -127 to 127
    short   upmove;         // -127 to 127
    vec3_t  angles;         // Viewing angles
} usercmd_t;
```

---

## 6. Console & Command Subsystem

| File | Path | Purpose |
|------|------|---------|
| `cmd.c` | `Quake/WinQuake/` | Command buffer, tokenization, execution |
| `cmd.h` | `Quake/WinQuake/` | Command interface |
| `cvar.c` | `Quake/WinQuake/` | Console variable registration, get/set |
| `cvar.h` | `Quake/WinQuake/` | CVar structure definition |
| `console.c` | `Quake/WinQuake/` | Console rendering, scrollback, notification |
| `console.h` | `Quake/WinQuake/` | Console interface |

**Command execution pipeline:**
```
Cbuf_AddText("command args\n")     → Queue in command buffer
    ↓
Cbuf_Execute()                      → Process buffer each frame
    ↓
Cmd_TokenizeString("command args")  → Parse into argc/argv
    ↓
Cmd_ExecuteString("command")        → Look up and execute
    ↓
xcommand_t callback()               → Registered function runs
```

**CVar structure (`Quake/WinQuake/cvar.h`):**
```
typedef struct cvar_s {
    char     *name;      // Variable name
    char     *string;    // String value
    qboolean  archive;   // Save to config.cfg
    qboolean  server;    // Send to clients
    float     value;     // Numeric value (auto-parsed)
    struct cvar_s *next; // Linked list pointer
} cvar_t;
```

---

## 7. Platform Abstraction Layer

| File | Path | Platform |
|------|------|----------|
| `sys_win.c` | `Quake/WinQuake/` | Windows (Win32 API, console allocation, timer) |
| `sys_linux.c` | `Quake/WinQuake/` | Linux (signal handling, `/dev/` access) |
| `sys_dos.c` | `Quake/WinQuake/` | MS-DOS (real-mode, DPMI, interrupt handlers) |
| `sys_sun.c` | `Quake/WinQuake/` | Solaris/SunOS |
| `sys_null.c` | `Quake/WinQuake/` | Null platform (stub for porting reference) |
| `sys.h` | `Quake/WinQuake/` | Platform interface definition |

**Interface contract (`Quake/WinQuake/sys.h`):**

| Function | Purpose |
|----------|---------|
| `Sys_FileOpenRead()` / `Sys_FileOpenWrite()` | File I/O |
| `Sys_FileRead()` / `Sys_FileWrite()` | File read/write |
| `Sys_FloatTime()` | High-resolution timer |
| `Sys_Error()` | Fatal error handler |
| `Sys_Printf()` | Console output |
| `Sys_Quit()` | Clean shutdown |
| `Sys_SendKeyEvents()` | Pump platform input events |
| `Sys_ConsoleInput()` | Read from stdin (dedicated server) |
| `Sys_MakeCodeWriteable()` | Make memory executable (for self-modifying ASM) |

---

## 8. Model/BSP Loading Subsystem

| File | Path | Purpose |
|------|------|---------|
| `model.c` | `Quake/WinQuake/` | Model loading, caching, BSP/alias/sprite format parsing |
| `model.h` | `Quake/WinQuake/` | Model data structures |
| `gl_model.c` | `Quake/WinQuake/` | OpenGL-specific model loading (texture upload) |
| `gl_model.h` | `Quake/WinQuake/` | GL model structures |
| `bspfile.h` | `Quake/WinQuake/` | BSP file format definition (version 29, 15 lumps) |

**BSP lumps (`Quake/WinQuake/bspfile.h`):**

| Lump | Content |
|------|---------|
| `LUMP_ENTITIES` | Entity definition strings |
| `LUMP_PLANES` | Splitting planes |
| `LUMP_TEXTURES` | Texture data (mip-mapped) |
| `LUMP_VERTEXES` | Vertex coordinates |
| `LUMP_VISIBILITY` | PVS (Potentially Visible Set) data |
| `LUMP_NODES` | BSP tree nodes |
| `LUMP_TEXINFO` | Texture alignment info |
| `LUMP_FACES` | Surface polygons |
| `LUMP_LIGHTING` | Lightmap data |
| `LUMP_CLIPNODES` | Collision hull nodes |
| `LUMP_LEAFS` | BSP tree leaves |
| `LUMP_MARKSURFACES` | Leaf-to-surface mapping |
| `LUMP_EDGES` | Edge definitions |
| `LUMP_SURFEDGES` | Surface-to-edge mapping |
| `LUMP_MODELS` | Brush model list |

**Model types supported:**

| Type | Format | Magic | Usage |
|------|--------|-------|-------|
| Brush | BSP v29 | — | World geometry, doors, platforms |
| Alias | MDL v6 | `IDPO` | Players, monsters, items |
| Sprite | SPR v1 | `IDSP` | Particles, effects, UI elements |

---

## 9. Video Subsystem

| File | Path | Platform | API |
|------|------|----------|-----|
| `vid_win.c` | `Quake/WinQuake/` | Windows | DirectDraw, DIB, MGL (Scitech) |
| `vid_dos.c` | `Quake/WinQuake/` | DOS | VGA registers, VESA BIOS |
| `vid_svgalib.c` | `Quake/WinQuake/` | Linux | SVGAlib |
| `vid_x.c` | `Quake/WinQuake/` | Linux/Unix | X11 (XImage/XShmImage) |
| `vid_sunx.c` | `Quake/WinQuake/` | Solaris | X11 |
| `vid_sunxil.c` | `Quake/WinQuake/` | Solaris | XIL (accelerated) |
| `vid_null.c` | `Quake/WinQuake/` | Any | Null driver (stub) |
| `vid.h` | `Quake/WinQuake/` | — | Video interface definition |

**Video state (`viddef_t` in `Quake/WinQuake/vid.h`):**

| Field | Purpose |
|-------|---------|
| `buffer` | Framebuffer pixel pointer |
| `colormap` | 8-bit color palette lookup |
| `rowbytes` | Bytes per scanline (pitch) |
| `width`, `height` | Current resolution |
| `aspect` | Pixel aspect ratio |
| `numpages` | Page-flip count (double/triple buffering) |

---

## 10. QuakeWorld-Specific Components

### 10.1 QW Forwarding Proxy

| File | Path | Purpose |
|------|------|---------|
| `qwfwd.c` | `Quake/QW/qwfwd/` | UDP relay proxy: forwards packets between clients and servers |
| `misc.c` | `Quake/QW/qwfwd/` | Utility functions for the proxy |

Architecture: Maintains a linked list of `peer_t` structures, each mapping a client address to a server connection. Uses `select()` for non-blocking I/O.

### 10.2 Gas2Masm Converter

| File | Path | Purpose |
|------|------|---------|
| `gas2masm.c` | `Quake/QW/gas2masm/` | Converts GNU as (AT&T syntax) x86 assembly to MASM (Intel syntax) |

Translates 100+ instruction mnemonics, register names, addressing modes, and section directives.

### 10.3 QuakeC Game Logic

| File | Path | Purpose |
|------|------|---------|
| `defs.qc` | `Quake/qw-qc/` | Global definitions, entity fields, builtin function declarations |
| `weapons.qc` | `Quake/qw-qc/` | 7 weapon implementations with projectile physics |
| `items.qc` | `Quake/qw-qc/` | Health, armor, ammo, keys, powerup pickups |
| `combat.qc` | `Quake/qw-qc/` | Damage system (`T_Damage`, `T_RadiusDamage`) |
| `client.qc` | `Quake/qw-qc/` | Player spawning, death, respawn, level transitions |
| `spectate.qc` | `Quake/qw-qc/` | Spectator mode (non-interactive observation) |
| `doors.qc` | `Quake/qw-qc/` | Door mechanics (linked, keyed, secret) |
| `triggers.qc` | `Quake/qw-qc/` | Trigger entities (teleport, hurt, push, relay, counter) |
| `plats.qc` | `Quake/qw-qc/` | Moving platforms and trains |
| `world.qc` | `Quake/qw-qc/` | World initialization and asset precaching |
| `player.qc` | `Quake/qw-qc/` | Player animation frame definitions |
| `buttons.qc` | `Quake/qw-qc/` | Pushbutton mechanics |
| `misc.qc` | `Quake/qw-qc/` | Lights, fireballs, explosive barrels |
| `subs.qc` | `Quake/qw-qc/` | Movement utilities and target-firing system |
| `server.qc` | `Quake/qw-qc/` | Monster waypoint pathing (path_corner) |
| `models.qc` | `Quake/qw-qc/` | Model entity spawning helpers |
| `sprites.qc` | `Quake/qw-qc/` | Sprite entity spawning |
