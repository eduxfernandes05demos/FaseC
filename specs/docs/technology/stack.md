# Technology Stack — Quake Engine

> Reverse-engineered from the id Software Quake source code (1996-1997).
> All file references point to actual source files in this repository.

---

## 1. Programming Languages

| Language | Usage | Files | Approximate LOC |
|----------|-------|-------|-----------------|
| **C (ANSI C / C89)** | Engine, server, client, tools | `legacy-src/desktop-engine/*.c`, `legacy-src/QW/**/*.c` | ~100,000 |
| **x86 Assembly (GAS)** | Performance-critical rendering, math | `legacy-src/desktop-engine/*.s` | ~5,000 |
| **x86 Assembly (MASM)** | Windows-specific assembly | Generated via `gas2masm` | — |
| **QuakeC** | Game logic scripting | `legacy-src/qw-qc/*.qc` | ~11,000 |

### C Language Details
- Standard: ANSI C (C89) with some compiler-specific extensions
- No C++ features used anywhere in the codebase
- Heavy use of global variables for engine state
- Function pointers for polymorphism (callbacks, drivers)
- `typedef struct` pattern for all data types

### x86 Assembly Details
- AT&T syntax (GNU as) for Linux builds: `legacy-src/desktop-engine/*.s`
- Intel syntax (MASM) for Windows via `gas2masm` converter: `legacy-src/QW/gas2masm/gas2masm.c`
- Used for: inner rendering loops, texture mapping, math operations, sound mixing
- Key files: `d_draw.s`, `d_draw16.s`, `d_parta.s`, `d_polysa.s`, `d_scana.s`, `d_varsa.s`, `d_copy.s`, `math.s`, `r_aclipa.s`, `r_aliasa.s`, `r_drawa.s`, `r_edgea.s`, `r_varsa.s`, `snd_mixa.s`, `surf8.s`, `surf16.s`, `worlda.s`, `sys_dosa.s`, `sys_wina.s`, `dosasm.s`

### QuakeC Details
- Custom scripting language designed by John Carmack
- C-like syntax with entity-field system
- Compiled to bytecode (`progs.dat` / `qwprogs.dat`) by external compiler (not included in this repo)
- ~80 opcodes, 3-operand instruction format
- Types: `float`, `vector` (3 floats), `string`, `entity`, `void`, function pointers
- No arrays, no dynamic allocation, no strings manipulation beyond literals

---

## 2. Platform APIs

### Windows

| API | Version | Usage | Source Files |
|-----|---------|-------|-------------|
| **Win32 API** | Windows 95+ | Window management, timers, file I/O | `legacy-src/desktop-engine/sys_win.c` |
| **DirectDraw** | DirectX 3-5 | Fullscreen video mode switching, page flipping | `legacy-src/desktop-engine/vid_win.c` |
| **DirectInput** | DirectX 3-5 | Mouse input in fullscreen | `legacy-src/desktop-engine/in_win.c` |
| **DirectSound** | DirectX 3-5 | Audio output via DMA-like buffer | `legacy-src/desktop-engine/snd_win.c` |
| **Winsock** | 1.1 | UDP networking | `legacy-src/desktop-engine/net_wins.c` |
| **Winsock (IPX)** | 1.1 | IPX/SPX networking | `legacy-src/desktop-engine/net_wipx.c` |
| **MCI** | Windows 3.1+ | CD audio music playback | `legacy-src/desktop-engine/cd_win.c` |
| **OpenGL** | 1.1 | Hardware-accelerated rendering | `legacy-src/desktop-engine/gl_vidnt.c` |
| **MGL (Scitech)** | 4.x | VESA/VBE video mode management | `legacy-src/desktop-engine/vid_win.c` |

### Linux

| API | Usage | Source Files |
|-----|-------|-------------|
| **POSIX** | File I/O, signals, process management | `legacy-src/desktop-engine/sys_linux.c` |
| **SVGAlib** | Direct framebuffer access (console mode) | `legacy-src/desktop-engine/vid_svgalib.c` |
| **X11 (Xlib)** | Windowed rendering, keyboard/mouse | `legacy-src/desktop-engine/vid_x.c` |
| **X11 SHM Extension** | Shared memory for fast blitting | `legacy-src/desktop-engine/vid_x.c` |
| **OSS** | Sound output via `/dev/dsp` | `legacy-src/desktop-engine/snd_linux.c` |
| **BSD Sockets** | UDP networking | `legacy-src/desktop-engine/net_udp.c` |
| **Mesa3D / GLX** | OpenGL rendering | `legacy-src/desktop-engine/gl_vidlinux.c`, `gl_vidlinuxglx.c` |
| **CD-ROM ioctl** | CD audio via `/dev/cdrom` | `legacy-src/desktop-engine/cd_linux.c` |

### DOS

| API | Usage | Source Files |
|-----|-------|-------------|
| **VGA Registers** | Direct hardware video access | `legacy-src/desktop-engine/vid_dos.c`, `vid_vga.c` |
| **VESA BIOS** | Extended video modes | `legacy-src/desktop-engine/vid_ext.c` |
| **DMA Controller** | Sound output | `legacy-src/desktop-engine/snd_dos.c` |
| **Gravis UltraSound** | GUS-specific sound | `legacy-src/desktop-engine/snd_gus.c` |
| **IPX** | DOS networking | `legacy-src/desktop-engine/net_ipx.c` |
| **Serial ports** | Modem multiplayer | `legacy-src/desktop-engine/net_ser.c`, `net_comx.c` |
| **DPMI** | Protected mode memory management | `legacy-src/desktop-engine/sys_dos.c` |
| **CWSDPMI** | DOS extender (included: `cwsdpmi.exe`) | `legacy-src/desktop-engine/cwsdpmi.exe` |

### Solaris/SunOS

| API | Usage | Source Files |
|-----|-------|-------------|
| **X11 (Xlib)** | Video output | `legacy-src/desktop-engine/vid_sunx.c` |
| **XIL** | Accelerated imaging library | `legacy-src/desktop-engine/vid_sunxil.c` |
| **Solaris audio** | Sound output | `legacy-src/desktop-engine/snd_sun.c` |
| **BSD Sockets** | Networking | `legacy-src/desktop-engine/net_udp.c` |
| **X11 input** | Keyboard/mouse | `legacy-src/desktop-engine/in_sun.c` |

---

## 3. Build Tools

| Tool | Platform | Usage | Build Files |
|------|----------|-------|-------------|
| **GCC** | Linux, Solaris | C compiler | `legacy-src/desktop-engine/Makefile.linuxi386`, `legacy-src/QW/Makefile.Linux` |
| **GNU as** | Linux | x86 assembler (AT&T syntax) | Assembly `.s` files |
| **GNU Make** | Linux, Solaris | Build automation | `Makefile.linuxi386`, `Makefile.Solaris`, `Makefile.Linux` |
| **MSVC 4.x/5.x** | Windows | C compiler + linker | `WinQuake.dsp`, `WinQuake.dsw`, `qw.dsw` |
| **MASM** | Windows | x86 assembler (Intel syntax) | Via `gas2masm` conversion |
| **QuakeC Compiler** | Any | Compiles `.qc` to `progs.dat` | `legacy-src/qw-qc/progs.src` (not included in repo) |

### Compiler Flags

**GCC Release** (`legacy-src/desktop-engine/Makefile.linuxi386`):
```
-O6 -m486 -ffast-math -funroll-loops -fomit-frame-pointer -fexpensive-optimizations
```

**GCC Debug**:
```
-g
```

**MSVC Release** (`legacy-src/desktop-engine/WinQuake.dsp`):
```
/G5 /GX /Ox /Ot /Og /Ob2
```

---

## 4. Data Formats

| Format | Extension | Usage | Defined In |
|--------|-----------|-------|------------|
| **BSP v29** | `.bsp` | Map/level geometry, visibility, lighting | `legacy-src/desktop-engine/bspfile.h` |
| **MDL v6** | `.mdl` | Animated character/item models (alias models) | `legacy-src/desktop-engine/modelgen.h`, `model.c` |
| **SPR v1** | `.spr` | Billboard sprites for effects | `legacy-src/desktop-engine/spritegn.h`, `model.c` |
| **WAD2** | `.wad` | Texture/graphic archives | `legacy-src/desktop-engine/wad.c`, `wad.h` |
| **WAV** | `.wav` | Sound effects | `legacy-src/desktop-engine/snd_mem.c` |
| **LMP** | `.lmp` | Lump graphics (HUD elements) | `legacy-src/desktop-engine/draw.c` |
| **DEM** | `.dem` | Demo recordings (gameplay playback) | `legacy-src/desktop-engine/cl_demo.c` |
| **PAK** | `.pak` | Archive files containing game assets | `legacy-src/desktop-engine/common.c` |
| **progs.dat** | `.dat` | Compiled QuakeC bytecode | `legacy-src/desktop-engine/pr_edict.c` |
| **config.cfg** | `.cfg` | Saved key bindings and cvar values | `legacy-src/desktop-engine/host_cmd.c` |

---

## 5. Network Protocols

| Protocol | Port | Usage | Source |
|----------|------|-------|--------|
| **Quake Protocol** (v15) | 26000 | WinQuake client-server | `legacy-src/desktop-engine/protocol.h` |
| **QuakeWorld Protocol** (v28) | 27500 (server), 27001 (client), 27000 (master) | QW internet multiplayer | `legacy-src/QW/client/protocol.h` |
| **UDP** | — | Primary transport | `net_udp.c`, `net_wins.c` |
| **IPX/SPX** | — | LAN transport (Windows/DOS) | `net_wipx.c`, `net_ipx.c` |
| **Serial** | COM1-4 | Modem-to-modem | `net_ser.c`, `net_comx.c` |

---

## 6. Third-Party Libraries / SDKs

| Library | Version | Usage | Location |
|---------|---------|-------|----------|
| **DirectX SDK** | 3-5 | Windows DirectDraw/DirectInput/DirectSound headers | `legacy-src/desktop-engine/dxsdk/`, `legacy-src/QW/dxsdk/` |
| **Scitech MGL** | 4.x | Multi-platform graphics library (VESA/VBE) | `legacy-src/desktop-engine/scitech/`, `legacy-src/QW/scitech/` |
| **3Dfx SDK** | — | 3Dfx Voodoo MiniGL headers | `legacy-src/desktop-engine/3dfx.txt` |
| **CWSDPMI** | — | DOS protected mode interface | `legacy-src/desktop-engine/cwsdpmi.exe` |

> **Note**: No standard C library beyond libc, no STL, no external frameworks. The engine is almost entirely self-contained.

---

## 7. Development & Source Control

| Aspect | Detail |
|--------|--------|
| **Original VCS** | Not included (id Software used internal tools) |
| **Current VCS** | Git (this repository) |
| **License** | GPL v2 (released 1999) |
| **IDE Support** | MSVC 4.x/5.x workspace files (`.dsw`, `.dsp`) |
| **Code Style** | K&R-style C, tabs for indentation, `//` comments rare |
| **Documentation** | Sparse inline comments; no API docs |
| **Testing** | No automated tests exist in the codebase |
