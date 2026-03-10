# Product Requirements Document — Quake Engine

> **Reverse-engineered from the id Software Quake source code (1996-1997)**
> This document synthesizes the product vision and requirements based solely on the existing codebase.

---

## 1. Product Overview

**Quake** is a first-person shooter (FPS) game engine and game developed by id Software, released in 1996. The codebase contains:

| Deliverable | Description |
|-------------|-------------|
| **WinQuake** | Full single-player and LAN multiplayer engine with software and OpenGL rendering |
| **QuakeWorld** | Internet-optimized multiplayer client and dedicated server with client-side prediction |
| **QuakeC Game Logic** | Interpreted scripting layer defining all gameplay mechanics |

The engine pioneered real-time 3D rendering with true 3D environments (BSP-based), client-server networked multiplayer, and a bytecode-interpreted game logic scripting language.

---

## 2. Target Platforms

| Platform | Client | Server | Renderer |
|----------|--------|--------|----------|
| **MS-DOS** | ✅ | ✅ | Software (VGA/VESA) |
| **Windows 95/NT** | ✅ | ✅ | Software (DirectDraw/DIB) + OpenGL |
| **Linux (i386)** | ✅ | ✅ | Software (SVGAlib/X11) + OpenGL (Mesa/GLX) |
| **Solaris (Sparc/x86)** | ✅ | ✅ | Software (X11/XIL) |
| **3Dfx Voodoo** | ✅ | — | OpenGL (MiniGL/3dfxgl) |

> **Limitation**: No macOS, no ARM, no 64-bit support. Assembly optimizations are x86-only.

---

## 3. Core Features

### 3.1 Rendering Engine

- **Software renderer**: Span-based rasterization, edge sorting, affine texture mapping, lightmaps, mip-mapping
- **OpenGL renderer**: Hardware-accelerated polygon rendering, multitexture lightmaps, dynamic lights
- **BSP tree traversal**: Front-to-back rendering with PVS (Potentially Visible Set) culling
- **Alias model rendering**: Animated mesh models (MDL format) for characters and items
- **Sprite rendering**: Billboard sprites for particles and effects
- **Sky rendering**: Scrolling sky texture with parallax layers
- **Particle system**: Point-based particle effects for impacts, blood, explosions
- **Console overlay**: Drop-down developer console with command input

> Source: `Quake/WinQuake/r_main.c`, `Quake/WinQuake/gl_rmain.c`, `Quake/WinQuake/r_bsp.c`

### 3.2 Networking

#### WinQuake (LAN-focused)
- Reliable and unreliable message channels
- Loopback, UDP, IPX, and serial modem drivers
- Sequence-numbered reliable delivery with ACK retransmission
- Entity state snapshot broadcasting

> Source: `Quake/WinQuake/net_main.c`, `Quake/WinQuake/net_dgrm.c`, `Quake/WinQuake/protocol.h`

#### QuakeWorld (Internet-optimized)
- **Delta compression**: Only transmit changed entity fields using bitmask flags
- **Client-side prediction**: Run player physics locally to mask latency
- **Dedicated server**: Standalone server without rendering overhead
- **Spectator mode**: Non-interactive observation with player tracking
- **Master server registration**: Servers register with central master for discovery
- **Anti-cheat**: Model checksums, download restrictions, choke counting

> Source: `Quake/QW/client/cl_pred.c`, `Quake/QW/client/cl_ents.c`, `Quake/QW/server/sv_ents.c`

### 3.3 QuakeC Scripting Engine

- Bytecode virtual machine with ~80 opcodes (3-operand format)
- ~100 built-in functions bridging script to engine (traceline, sound, spawn, etc.)
- Entity-field system with think/touch/use/blocked callbacks
- Compiled from `.qc` source files into `progs.dat` / `qwprogs.dat`

> Source: `Quake/WinQuake/pr_exec.c`, `Quake/WinQuake/pr_edict.c`, `Quake/WinQuake/pr_cmds.c`

### 3.4 Sound System

- DMA-based circular buffer audio playback
- 3D spatialization with distance attenuation and stereo panning
- Multi-channel mixing (ambient + effects + music)
- CD audio music support
- Platform backends: Windows (DirectSound/waveOut), Linux (OSS `/dev/dsp`), DOS (DMA/GUS), Sun (audio device)

> Source: `Quake/WinQuake/snd_dma.c`, `Quake/WinQuake/snd_mix.c`, `Quake/WinQuake/sound.h`

### 3.5 Input System

- Keyboard with configurable key bindings
- Mouse with sensitivity and inversion support
- Joystick with 6-axis mapping (Windows)
- DirectInput support on Windows
- Platform-specific drivers: Windows, DOS, Sun, null

> Source: `Quake/WinQuake/in_win.c`, `Quake/WinQuake/keys.c`, `Quake/WinQuake/input.h`

### 3.6 Console & Command System

- Drop-down in-game console for developer interaction
- Command buffer with text-based command execution
- Console variables (cvars) for runtime configuration
- Key binding system mapping keys to commands
- Command aliasing and scripting via `alias` command

> Source: `Quake/WinQuake/cmd.c`, `Quake/WinQuake/cvar.c`, `Quake/WinQuake/console.c`

### 3.7 Game Logic (QuakeC)

- **7 weapons**: Axe, Shotgun, Super Shotgun, Nailgun, Super Nailgun, Grenade Launcher, Rocket Launcher, Lightning Gun
- **Items**: Health (small/mega), Armor (green/yellow/red), Ammo (shells/nails/rockets/cells), Keys (gold/silver), Powerups (Quad Damage, Invulnerability, Invisibility, Biosuit)
- **Map entities**: Doors (linked, keyed, secret), Buttons, Platforms, Trains, Triggers (multiple, once, teleport, hurt, push, relay, counter, secret)
- **Player mechanics**: Spawning, respawning, death, intermission, level transitions
- **Spectator mode**: Non-interactive observer with spawn-point teleportation
- **Damage system**: Armor reduction, quad multiplier, knockback, radius damage, team protection

> Source: `Quake/qw-qc/weapons.qc`, `Quake/qw-qc/items.qc`, `Quake/qw-qc/combat.qc`, `Quake/qw-qc/client.qc`

### 3.8 Map/BSP System

- BSP version 29 file format with 15 data lumps
- Brush model loading (world geometry)
- Alias model loading (animated characters — MDL format)
- Sprite model loading (billboard effects)
- Lazy-load caching with hunk memory allocation
- PVS (Potentially Visible Set) precomputed visibility data
- Collision detection via clip nodes (multiple hull sizes)

> Source: `Quake/WinQuake/model.c`, `Quake/WinQuake/bspfile.h`, `Quake/WinQuake/model.h`

### 3.9 Demo Recording & Playback

- Record gameplay to `.dem` files
- Play back recorded demos with full rendering
- Time-demo mode for benchmarking

> Source: `Quake/WinQuake/cl_demo.c`, `Quake/QW/client/cl_demo.c`

### 3.10 Save/Load System

- Save complete game state to disk (single-player)
- Restore game state including entity positions, health, inventory
- Quick-save and quick-load key bindings

> Source: `Quake/WinQuake/host_cmd.c`

---

## 4. Architecture Summary

```
┌──────────────────────────────────────────────────────┐
│                    Main Loop (host.c)                 │
├──────────┬──────────┬───────────┬────────────────────┤
│  Input   │  Server  │  Client   │     Renderer       │
│ (keys,   │ (physics,│ (parse,   │  (software / GL)   │
│  mouse)  │  QuakeC) │  predict) │                    │
├──────────┴──────────┴───────────┴────────────────────┤
│    Network    │    Sound    │    Console/Commands     │
├───────────────┴─────────────┴────────────────────────┤
│           Platform Abstraction Layer                  │
│     sys_*.c    vid_*.c    in_*.c    snd_*.c          │
└──────────────────────────────────────────────────────┘
```

---

## 5. Known Limitations & Gaps

| Area | Limitation |
|------|-----------|
| **Testing** | No automated test suite exists in the codebase |
| **Documentation** | Original code comments are sparse; no API documentation |
| **Build system** | Platform-specific Makefiles and MSVC `.dsp` projects; no unified build system |
| **64-bit support** | Not supported; pointer assumptions and assembly are 32-bit only |
| **Modern OS support** | No macOS, no modern Windows APIs, no Wayland/Pipewire |
| **Security** | No input sanitization for network messages; buffer overflow potential in string handling |
| **AI** | No monster AI exists in the QuakeWorld QuakeC code (only player vs player) |
| **Accessibility** | No accessibility features (subtitles, colorblind modes, key remapping UI) |
| **Localization** | English-only; hardcoded strings throughout |

---

## 6. Stakeholders

| Role | Entity |
|------|--------|
| **Original Developer** | id Software (John Carmack, Michael Abrash, John Cash) |
| **License** | GPL v2 (released 1999) |
| **Target Users** | FPS gamers, modders, engine developers |
| **Community** | Active source port community (Quakespasm, vkQuake, etc.) |

---

## 7. Success Metrics (Inferred from Code)

- Maintain 30+ FPS on target hardware (Pentium 75 MHz for software, Pentium 166 for GL)
- Support 16+ simultaneous players in QuakeWorld multiplayer
- Sub-200ms perceived latency with client-side prediction
- Support custom game logic through QuakeC modding
- Cross-platform operation on DOS, Windows, Linux, and Solaris
