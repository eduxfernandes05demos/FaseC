# External Dependencies — Quake Engine

> Reverse-engineered from the id Software Quake source code (1996-1997).
> All file references point to actual source files in this repository.

---

## 1. Overview

The Quake engine has minimal external dependencies by design. The engine is largely self-contained, implementing its own memory management, math library, and data format parsers. External dependencies are primarily platform SDKs and hardware abstraction libraries.

---

## 2. Windows Dependencies

### 2.1 DirectX SDK

**Location in repo**: `Quake/WinQuake/dxsdk/`, `Quake/QW/dxsdk/`

| Component | Header | Library | Used By |
|-----------|--------|---------|---------|
| **DirectDraw** | `ddraw.h` | `ddraw.lib` | `Quake/WinQuake/vid_win.c` — Fullscreen video mode switching, page flipping |
| **DirectInput** | `dinput.h` | `dinput.lib` | `Quake/WinQuake/in_win.c` — Mouse input in exclusive mode |
| **DirectSound** | `dsound.h` | `dsound.lib` | `Quake/WinQuake/snd_win.c` — Primary/secondary sound buffer management |
| **DirectX GUIDs** | — | `dxguid.lib` | GUID definitions for COM interface initialization |

**Version**: DirectX 3–5 era headers (1996-1997)

**Linker references** (`Quake/WinQuake/WinQuake.dsp`):
```
dxguid.lib
```

### 2.2 Scitech MGL (Multi-platform Graphics Library)

**Location in repo**: `Quake/WinQuake/scitech/`, `Quake/QW/scitech/`

| Component | Usage |
|-----------|-------|
| **MGL headers** | `Quake/WinQuake/scitech/include/` — Graphics mode enumeration |
| **MGL library** | `mgllt.lib` — Video mode switching, VESA/VBE support |

**Used by**: `Quake/WinQuake/vid_win.c` — Provides video mode enumeration and switching on Windows, especially for VESA-compatible graphics cards.

**Linker references** (`Quake/WinQuake/WinQuake.dsp`):
```
mgllt.lib
```

### 2.3 Windows System Libraries

| Library | Usage | Used By |
|---------|-------|---------|
| `kernel32.lib` | Process, memory, file I/O | `Quake/WinQuake/sys_win.c` |
| `user32.lib` | Window management, input | `Quake/WinQuake/sys_win.c`, `vid_win.c` |
| `gdi32.lib` | DIB (Device Independent Bitmap) rendering | `Quake/WinQuake/vid_win.c` |
| `winmm.lib` | Multimedia timers, joystick, MCI (CD audio) | `Quake/WinQuake/in_win.c`, `cd_win.c` |
| `wsock32.lib` | Winsock 1.1 networking | `Quake/WinQuake/net_wins.c`, `net_wipx.c` |
| `opengl32.lib` | OpenGL 1.1 API | `Quake/WinQuake/gl_vidnt.c` |
| `glu32.lib` | OpenGL Utility Library | `Quake/WinQuake/gl_vidnt.c` |
| `comctl32.lib` | Common controls (UI dialogs) | `Quake/WinQuake/conproc.c` |

---

## 3. Linux Dependencies

### 3.1 System Libraries

| Library | Flag | Usage | Used By |
|---------|------|-------|---------|
| **libc** | — | Standard C library | All source files |
| **libm** | `-lm` | Math functions (sin, cos, sqrt, etc.) | `Quake/WinQuake/mathlib.c` |
| **libX11** | `-lX11` | X Window System | `Quake/WinQuake/vid_x.c`, `in_sun.c` |
| **libXext** | `-lXext` | X11 shared memory extension (XShm) | `Quake/WinQuake/vid_x.c` |
| **libXxf86dga** | `-lXxf86dga` | XFree86 Direct Graphics Access | `Quake/WinQuake/vid_x.c` |
| **libvga** | `-lvga` | SVGAlib direct framebuffer access | `Quake/WinQuake/vid_svgalib.c` |

**Referenced in**: `Quake/WinQuake/Makefile.linuxi386`

### 3.2 OpenGL Libraries (Linux)

| Library | Flag | Usage | Used By |
|---------|------|-------|---------|
| **Mesa3D** | `-lMesaGL` | Software OpenGL implementation | `Quake/WinQuake/gl_vidlinux.c` |
| **GLX** | `-lGL` | X11 OpenGL extension | `Quake/WinQuake/gl_vidlinuxglx.c` |
| **3Dfx GL** | `-l3dfxgl` | 3Dfx Voodoo MiniGL | `Quake/WinQuake/Makefile.linuxi386` |

**Mesa include path**: `-I/usr/local/src/Mesa-3.0/include`
**Referenced in**: `Quake/WinQuake/Makefile.linuxi386`

### 3.3 Linux Sound

| Interface | Device | Usage | Used By |
|-----------|--------|-------|---------|
| **OSS (Open Sound System)** | `/dev/dsp` | Audio output | `Quake/WinQuake/snd_linux.c` |
| **CD-ROM ioctl** | `/dev/cdrom` | CD audio music | `Quake/WinQuake/cd_linux.c` |

> **Note**: No ALSA, PulseAudio, or PipeWire support — these did not exist in 1996-1997.

---

## 4. Solaris Dependencies

| Library | Flag | Usage | Used By |
|---------|------|-------|---------|
| **libm** | `-lm` | Math functions | All |
| **libsocket** | `-lsocket` | BSD socket networking | `Quake/WinQuake/net_udp.c` |
| **libnsl** | `-lnsl` | Name service lookup | Network code |
| **libX11** | `-lX11` | X11 video output | `Quake/WinQuake/vid_sunx.c` |
| **libxil** | `-lxil` | XIL imaging library (Sun accelerated) | `Quake/WinQuake/vid_sunxil.c` |

**Referenced in**: `Quake/WinQuake/Makefile.Solaris`, `Quake/QW/Makefile.Solaris`

---

## 5. DOS Dependencies

| Dependency | Usage | Used By |
|------------|-------|---------|
| **CWSDPMI** | DOS Protected Mode Interface (DPMI host) | `Quake/WinQuake/cwsdpmi.exe` |
| **DJGPP** | GCC cross-compiler for DOS | Build toolchain (not in repo) |
| **VGA hardware** | Direct register access for video | `Quake/WinQuake/vid_vga.c`, `vid_dos.c` |
| **DMA controller** | Direct Memory Access for sound | `Quake/WinQuake/snd_dos.c` |
| **Gravis UltraSound SDK** | GUS-specific sound card support | `Quake/WinQuake/snd_gus.c` |
| **IPX TSR** | Novell IPX protocol stack | `Quake/WinQuake/net_ipx.c` |
| **COM ports** | Serial modem I/O | `Quake/WinQuake/net_comx.c`, `net_ser.c` |

---

## 6. 3Dfx / Voodoo Dependencies

| Dependency | Usage | Source |
|------------|-------|--------|
| **3Dfx MiniGL** | OpenGL subset for Voodoo cards | `Quake/WinQuake/3dfx.txt` |
| **3dfxgl shared library** | Linux 3Dfx OpenGL | `Quake/QW/glqwcl.3dfxgl` |

**Build target**: `glquake.3dfxgl` in `Quake/WinQuake/Makefile.linuxi386`

---

## 7. Build-Time Dependencies

| Tool | Platform | Purpose |
|------|----------|---------|
| **GCC 2.x** | Linux/Solaris | C compiler |
| **GNU Make** | Linux/Solaris | Build automation |
| **GNU as** | Linux | x86 assembler |
| **MSVC 4.x/5.x** | Windows | C compiler, linker, resource compiler |
| **MASM** | Windows | x86 assembler |
| **gas2masm** | Windows | AT&T → MASM assembly converter (`Quake/QW/gas2masm/gas2masm.c`) |
| **QuakeC compiler (qcc)** | Any | Compiles `.qc` → `progs.dat` (not included in repo) |

---

## 8. Runtime Dependencies

| Dependency | Platform | Required For |
|------------|----------|-------------|
| **Game data files** (`pak0.pak`, `pak1.pak`) | All | Textures, models, sounds, maps |
| **OpenGL ICD** | Windows/Linux | Hardware-accelerated rendering |
| **SVGAlib** | Linux | Console-mode rendering |
| **X Server** | Linux/Solaris | Windowed rendering |
| **Sound device** (`/dev/dsp`) | Linux | Audio output |
| **CD-ROM drive** | All | Music playback (optional) |

---

## 9. Dependency Diagram

```mermaid
graph TD
    subgraph "Quake Engine"
        ENGINE[Engine Core<br/>C + x86 ASM]
        QC[QuakeC VM]
        SW[Software Renderer]
        GLREN[OpenGL Renderer]
    end

    subgraph "Windows SDKs"
        DX[DirectX SDK<br/>DirectDraw, DirectInput,<br/>DirectSound]
        MGL[Scitech MGL<br/>Video modes]
        WIN32[Win32 API<br/>kernel32, user32, gdi32]
        WSOCK[Winsock 1.1]
        OGL_W[opengl32.dll]
    end

    subgraph "Linux Libraries"
        X11[X11 / Xlib]
        SVGA[SVGAlib]
        MESA[Mesa3D / GLX]
        OSS[OSS /dev/dsp]
        POSIX[POSIX / libc]
    end

    subgraph "DOS"
        DPMI[CWSDPMI]
        VGA[VGA Registers]
        DMA[DMA Controller]
    end

    subgraph "Hardware"
        VOODOO[3Dfx Voodoo<br/>MiniGL]
        GUS[Gravis UltraSound]
    end

    ENGINE --> WIN32
    ENGINE --> POSIX
    ENGINE --> DPMI
    SW --> DX
    SW --> MGL
    SW --> SVGA
    SW --> X11
    SW --> VGA
    GLREN --> OGL_W
    GLREN --> MESA
    GLREN --> VOODOO
    ENGINE --> WSOCK
    ENGINE --> OSS
    ENGINE --> DMA
    ENGINE --> GUS
```

---

## 10. Notable Absences

The following common dependencies are **not used** in the codebase:

| Dependency | Reason |
|------------|--------|
| **SDL** | Did not exist in 1996 (SDL 1.0 released 1998) |
| **ALSA / PulseAudio / PipeWire** | Did not exist; OSS was the standard |
| **Wayland** | Did not exist; X11 was the only option |
| **Vulkan** | Did not exist (Vulkan 1.0 released 2016) |
| **OpenAL** | Did not exist; custom mixing was the norm |
| **zlib** | Not used; custom PAK archive format |
| **libpng / libjpeg** | Not used; custom WAD/LMP texture formats |
| **Any C++ STL** | Pure C codebase |
| **Any unit test framework** | No testing infrastructure exists |
