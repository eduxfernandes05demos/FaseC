# Task: Build System

> Technical implementation specification with file references.

---

## Subsystem Overview

The Quake codebase uses platform-specific build systems: GNU Makefiles for Linux/Solaris and MSVC project files for Windows. There is no unified cross-platform build system.

---

## File Inventory

### Linux Build Files

| File | Purpose |
|------|---------|
| `legacy-src/desktop-engine/Makefile.linuxi386` | WinQuake Linux/x86 build |
| `legacy-src/desktop-engine/Makefile.Solaris` | WinQuake Solaris build |
| `legacy-src/QW/Makefile.Linux` | QuakeWorld Linux build (client + server) |
| `legacy-src/QW/Makefile.Solaris` | QuakeWorld Solaris server build |

### Windows Build Files

| File | Purpose |
|------|---------|
| `legacy-src/desktop-engine/WinQuake.dsp` | MSVC project (WinQuake) |
| `legacy-src/desktop-engine/WinQuake.dsw` | MSVC workspace (WinQuake) |
| `legacy-src/desktop-engine/WinQuake.mdp` | MSVC makefile project |
| `legacy-src/QW/qw.dsw` | MSVC workspace (QuakeWorld all projects) |
| `legacy-src/QW/gas2masm/gas2masm.dsp` | MSVC project (gas2masm tool) |

### Utility Scripts

| File | Purpose |
|------|---------|
| `legacy-src/desktop-engine/clean.bat` | Windows build cleanup |
| `legacy-src/QW/clean.bat` | QW Windows build cleanup |
| `legacy-src/desktop-engine/q.bat`, `qa.bat`, `qb.bat`, `qt.bat`, `wq.bat` | Windows launch scripts |
| `legacy-src/desktop-engine/makezip.bat` | Package distribution |
| `legacy-src/QW/makezip.bat` | QW package distribution |

### Packaging Scripts

| File | Purpose |
|------|---------|
| `legacy-src/desktop-engine/quake.spec.sh` | Linux RPM spec for full Quake |
| `legacy-src/desktop-engine/quake-data.spec.sh` | Linux RPM spec for data files |
| `legacy-src/desktop-engine/quake-shareware.spec.sh` | Linux RPM spec for shareware |
| `legacy-src/desktop-engine/quake-hipnotic.spec.sh` | Linux RPM spec for Hipnotic expansion |
| `legacy-src/desktop-engine/quake-rogue.spec.sh` | Linux RPM spec for Rogue expansion |
| `legacy-src/QW/qwcl.spec.sh` | QW client RPM spec |
| `legacy-src/QW/qwcl.x11.spec.sh` | QW X11 client RPM spec |
| `legacy-src/QW/qwsv.spec.sh` | QW server RPM spec |
| `legacy-src/QW/glqwcl.spec.sh` | QW GL client RPM spec |

---

## WinQuake Linux Targets (`legacy-src/desktop-engine/Makefile.linuxi386`)

| Target | Binary | Renderer | Video | Sound |
|--------|--------|----------|-------|-------|
| `squake` | `squake` | Software | SVGAlib | OSS |
| `quake.x11` | `quake.x11` | Software | X11 | OSS |
| `glquake` | `glquake` | OpenGL | Mesa SVGA | OSS |
| `glquake.glx` | `glquake.glx` | OpenGL | X11 GLX | OSS |
| `glquake.3dfxgl` | `glquake.3dfxgl` | OpenGL | 3Dfx GL | OSS |

### Build Modes
- `make build_debug` — Debug symbols (`-g`)
- `make build_release` — Optimized (`-O6 -ffast-math -funroll-loops -fomit-frame-pointer`)

### Compiler Configuration
```makefile
CC = gcc
BASE_CFLAGS = -Did_ver -Did_rev
RELEASE_CFLAGS = $(BASE_CFLAGS) -O6 -m486 -ffast-math -funroll-loops -fomit-frame-pointer -fexpensive-optimizations
DEBUG_CFLAGS = $(BASE_CFLAGS) -g
```

---

## QuakeWorld Linux Targets (`legacy-src/QW/Makefile.Linux`)

| Target | Binary | Description |
|--------|--------|-------------|
| `qwsv` | `qwsv` | Dedicated server (all platforms) |
| `qwcl` | `qwcl` | Software client (SVGAlib) |
| `qwcl.x11` | `qwcl.x11` | Software client (X11) |
| `glqwcl` | `glqwcl` | OpenGL client (Mesa) |
| `glqwcl.glx` | `glqwcl.glx` | OpenGL client (GLX) |

---

## Windows Build Configurations (`legacy-src/desktop-engine/WinQuake.dsp`)

| Configuration | Description |
|---------------|-------------|
| Win32 Release | Optimized software renderer |
| Win32 Debug | Debug software renderer |
| Win32 GL Release | Optimized OpenGL renderer |
| Win32 GL Debug | Debug OpenGL renderer |

### MSVC Compiler Flags
```
/G5      — Optimize for Pentium Pro
/GX      — Enable C++ exception handling
/Ox      — Maximum optimization
/Ot      — Favor speed over size
```

### Linked Libraries
```
dxguid.lib          — DirectX GUIDs
mgllt.lib           — Scitech MGL
winmm.lib           — Windows multimedia
wsock32.lib         — Winsock networking
opengl32.lib        — OpenGL (GL builds only)
glu32.lib           — GLU (GL builds only)
kernel32.lib user32.lib gdi32.lib — Standard Win32
```

---

## Solaris Build (`legacy-src/desktop-engine/Makefile.Solaris`)

| Target | Description |
|--------|-------------|
| `quake.sw` | Software X11 renderer |
| `quake.xil` | XIL-accelerated renderer |

```makefile
LIBS = -lm -lX11
XIL_LIBS = -lm -lX11 -lxil
```

---

## QuakeC Compilation

| File | Purpose |
|------|---------|
| `legacy-src/qw-qc/progs.src` | Lists `.qc` files in compilation order |
| `legacy-src/qw-qc/qwprogs.dat` | Pre-compiled bytecode (included in repo) |

**Compiler**: `qcc` (not included in repository)

---

## Object File Count (~90 per target)

### Core Objects (all targets)
```
common.o zone.o cmd.o cvar.o console.o
mathlib.o crc.o wad.o
```

### Rendering Objects
```
Software: r_main.o r_bsp.o r_edge.o r_surf.o r_alias.o r_light.o r_part.o r_sky.o r_sprite.o
          d_edge.o d_scan.o d_surf.o d_init.o d_sky.o d_sprite.o d_part.o d_fill.o d_zpoint.o
OpenGL:   gl_rmain.o gl_rsurf.o gl_mesh.o gl_rlight.o gl_draw.o gl_warp.o gl_model.o gl_refrag.o
```

### Server Objects
```
sv_main.o sv_move.o sv_phys.o sv_user.o
pr_cmds.o pr_edict.o pr_exec.o
world.o
```

### Client Objects
```
cl_main.o cl_demo.o cl_input.o cl_parse.o cl_tent.o
keys.o menu.o sbar.o
```

### Network Objects
```
WinQuake: net_main.o net_dgrm.o net_loop.o net_udp.o
QW: net_chan.o net_udp.o
```

---

## Limitations

- No unified build system (no CMake, autotools, or Meson)
- Windows builds require MSVC 4.x/5.x (no modern MSVC support)
- x86-only assembly (no ARM, x64, or other architectures)
- No dependency tracking in Makefiles (manual `make clean` needed)
- No automated testing in build process
- Package specs (.spec.sh) target RPM only (no DEB, no modern packaging)
