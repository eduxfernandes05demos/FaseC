# Task: Platform Abstraction Layer

> Technical implementation specification with file references.

---

## Subsystem Overview

The platform abstraction layer isolates the engine from OS-specific APIs through a common interface defined in header files. Each platform provides its own implementation, selected at compile time via Makefile object lists.

---

## File Inventory

### System Layer (`legacy-src/desktop-engine/`)

| File | Platform | Key APIs Used |
|------|----------|---------------|
| `sys_win.c` | Windows | Win32 API, performance counters, console allocation |
| `sys_linux.c` | Linux | POSIX, signals, `/proc` |
| `sys_dos.c` | DOS | DPMI, interrupts, real-mode |
| `sys_sun.c` | Solaris | POSIX, Solaris-specific |
| `sys_null.c` | Any | Stub (porting reference) |
| `sys.h` | — | Interface definition |

### Video Layer (`legacy-src/desktop-engine/`)

| File | Platform | API |
|------|----------|-----|
| `vid_win.c` | Windows | DirectDraw, DIB, MGL (Scitech) |
| `vid_dos.c` | DOS | VGA registers, VESA |
| `vid_svgalib.c` | Linux | SVGAlib |
| `vid_x.c` | Linux/Unix | X11 (XImage, XShm) |
| `vid_sunx.c` | Solaris | X11 |
| `vid_sunxil.c` | Solaris | XIL (accelerated) |
| `vid_vga.c` | DOS | VGA mode 13h |
| `vid_ext.c` | DOS | VESA BIOS extensions |
| `vid_null.c` | Any | Stub |
| `vid.h` / `vid_dos.h` | — | Interface definitions |

### Input Layer (`legacy-src/desktop-engine/`)

| File | Platform | API |
|------|----------|-----|
| `in_win.c` | Windows | DirectInput, Win32 |
| `in_dos.c` | DOS | Interrupt handlers |
| `in_sun.c` | Solaris | X11 events |
| `in_null.c` | Any | Stub |
| `input.h` | — | Interface definition |

### Sound Layer (`legacy-src/desktop-engine/`)

| File | Platform | API |
|------|----------|-----|
| `snd_win.c` | Windows | DirectSound |
| `snd_linux.c` | Linux | OSS `/dev/dsp` |
| `snd_sun.c` | Solaris | Sun audio |
| `snd_dos.c` | DOS | DMA controller |
| `snd_gus.c` | DOS | Gravis UltraSound |
| `snd_next.c` | NeXTSTEP | NeXT audio |
| `snd_null.c` | Any | Stub |

### CD Audio Layer (`legacy-src/desktop-engine/`)

| File | Platform | API |
|------|----------|-----|
| `cd_win.c` | Windows | MCI |
| `cd_linux.c` | Linux | CD-ROM ioctl |
| `cd_null.c` | Any | Stub |

---

## System Interface (`sys.h`)

| Function | Purpose | Platform Differences |
|----------|---------|---------------------|
| `Sys_FloatTime()` | High-resolution timer | Win: `QueryPerformanceCounter`, Linux: `gettimeofday`, DOS: timer interrupt |
| `Sys_Error()` | Fatal error | Win: MessageBox, Linux: fprintf+exit, DOS: text mode+exit |
| `Sys_Printf()` | Console output | Win: console window, Linux: printf, DOS: printf |
| `Sys_Quit()` | Clean exit | Platform-specific cleanup |
| `Sys_FileOpenRead/Write()` | File I/O | Standard C `fopen` with path translation |
| `Sys_SendKeyEvents()` | Pump input | Win: MSG loop, Linux: X events, DOS: IRQ poll |
| `Sys_ConsoleInput()` | Read stdin | Dedicated server console input |
| `Sys_MakeCodeWriteable()` | Self-modifying code | `mprotect` or VirtualProtect |

---

## Build System Integration

### Linux Makefile (`legacy-src/desktop-engine/Makefile.linuxi386`)
```makefile
# Software SVGA
SQUAKE_OBJS = ... sys_linux.o vid_svgalib.o in_null.o snd_linux.o cd_linux.o ...

# Software X11
X11_OBJS = ... sys_linux.o vid_x.o in_null.o snd_linux.o cd_linux.o ...

# OpenGL
GLQUAKE_OBJS = ... sys_linux.o gl_vidlinux.o in_null.o snd_linux.o cd_linux.o ...
```

### Windows Project (`legacy-src/desktop-engine/WinQuake.dsp`)
```
Source: sys_win.c, vid_win.c, in_win.c, snd_win.c, cd_win.c, conproc.c
```

### Solaris Makefile (`legacy-src/desktop-engine/Makefile.Solaris`)
```makefile
OBJS = ... sys_sun.o vid_sunx.o in_sun.o snd_sun.o cd_null.o ...
```

---

## Porting Guide

To port to a new platform:
1. Implement `sys_<platform>.c` with all functions from `sys.h`
2. Implement `vid_<platform>.c` with all functions from `vid.h`
3. Implement `in_<platform>.c` with all functions from `input.h`
4. Implement `snd_<platform>.c` with `SNDDMA_Init/Submit/Shutdown`
5. Implement `cd_<platform>.c` with CD audio functions (or use `cd_null.c`)
6. Create a Makefile or project file selecting the new platform files
7. Handle x86 assembly (provide C fallbacks if not x86)

The `*_null.c` files serve as templates showing the minimum interface that must be implemented.
