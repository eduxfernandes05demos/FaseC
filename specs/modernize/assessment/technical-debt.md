# Technical Debt Inventory — Quake Engine

> Comprehensive inventory of technical debt for modernization planning.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

The Quake engine carries significant technical debt typical of a 1996-era C application optimized for performance on limited hardware. The debt spans unsafe coding patterns, platform-specific implementations, missing abstractions, and hard-coded configuration values. This inventory catalogs **6 major debt categories** with specific file references for remediation planning.

---

## 2. Unsafe C Patterns

### 2.1 Unbounded String Operations

The codebase contains **664 instances** of unsafe string functions:

| Function | Count | Risk | Replacement |
|----------|-------|------|-------------|
| `sprintf` / `vsprintf` | 362 | Buffer overflow | `snprintf` / `vsnprintf` |
| `strcpy` / `Q_strcpy` | 232 | Buffer overflow | `strncpy` or `strlcpy` |
| `strcat` / `Q_strcat` | 70 | Buffer overflow | `strncat` or `strlcat` |

**High-priority files** (most instances):

| File | sprintf | strcpy | strcat | Total |
|------|---------|--------|--------|-------|
| `Quake/WinQuake/common.c` | 7 | 12 | 1 | 20 |
| `Quake/WinQuake/host_cmd.c` | 5+ | 3+ | 8 | 16+ |
| `Quake/WinQuake/net_dgrm.c` | 3+ | 5+ | 1 | 9+ |
| `Quake/WinQuake/cmd.c` | 2 | 2+ | 6 | 10+ |
| `Quake/QW/server/sv_main.c` | 5+ | 3+ | 3 | 11+ |
| `Quake/QW/client/cl_main.c` | 5+ | 3+ | 5 | 13+ |
| `Quake/WinQuake/console.c` | 4 | 1+ | 0 | 5+ |
| `Quake/WinQuake/gl_model.c` | 4 | 3+ | 0 | 7+ |

### 2.2 Unchecked Memory Allocations

**9+ instances** of `malloc()` without NULL checks:

| File | Line | Code |
|------|------|------|
| `Quake/WinQuake/host.c` | 791 | `com_argv = malloc(com_argc * sizeof(char *));` |
| `Quake/WinQuake/host.c` | 796 | `p = malloc(len);` |
| `Quake/WinQuake/gl_warp.c` | 410 | `pcx_rgb = malloc(count * 4);` |
| `Quake/WinQuake/gl_warp.c` | 520 | `targa_rgba = malloc(numPixels*4);` |
| `Quake/WinQuake/gl_screen.c` | 618 | `buffer = malloc(glwidth*glheight*3 + 18);` |
| `Quake/QW/client/gl_screen.c` | 651 | `buffer = malloc(glwidth*glheight*3 + 18);` |
| `Quake/QW/client/gl_screen.c` | 879 | `newbuf = malloc(glheight * glwidth * 3);` |
| `Quake/QW/client/cl_parse.c` | 490 | `upload_data = malloc(size);` |
| `Quake/QW/client/screen.c` | 845 | `newbuf = malloc(w*h);` |
| `Quake/QW/qwfwd/qwfwd.c` | 225 | `p = malloc(sizeof *p);` |

**Only 1 correct NULL check found**: `Quake/QW/server/sys_unix.c:235`

### 2.3 Unsafe System Calls

| File | Line | Code | Risk |
|------|------|------|------|
| `Quake/WinQuake/sys_linux.c` | 276 | `system(cmd);` | Command injection |
| `Quake/QW/client/sys_linux.c` | 278 | `system(cmd);` | Command injection |

---

## 3. Platform-Specific Code

### 3.1 Conditional Compilation Spread

**23 files** use platform conditionals instead of abstraction layers:

| Category | Platform Files | Abstraction Level |
|----------|---------------|-------------------|
| System | `sys_win.c`, `sys_linux.c`, `sys_dos.c`, `sys_sun.c` | None — compile-time selection |
| Video | `vid_win.c`, `vid_svgalib.c`, `vid_x.c`, `vid_dos.c`, `vid_null.c` | None |
| Input | `in_win.c`, `in_dos.c`, `in_sun.c`, `in_null.c` | None |
| Sound | `snd_win.c`, `snd_linux.c`, `snd_dos.c`, `snd_sun.c`, `snd_null.c` | None |
| CD Audio | `cd_win.c`, `cd_linux.c`, `cd_null.c` | None |
| Network | `net_wins.c`, `net_udp.c`, `net_wipx.c`, `net_ipx.c` | Partial (driver struct) |
| OpenGL | `gl_vidnt.c`, `gl_vidlinux.c`, `gl_vidlinuxglx.c` | None |

### 3.2 #ifdef Scattering

Platform conditionals embedded in shared code:

| File | Line | Conditional |
|------|------|-------------|
| `Quake/WinQuake/host.c` | 908 | `#ifdef _WIN32` |
| `Quake/WinQuake/snd_mix.c` | 51-63 | `#ifdef _WIN32` |
| `Quake/WinQuake/snd_dma.c` | 196+ | `#ifdef _WIN32` |
| `Quake/WinQuake/mathlib.c` | various | `#ifdef id386` |
| `Quake/WinQuake/glquake.h` | various | `#ifdef _WIN32` |

---

## 4. Missing Abstractions

### 4.1 Global State (105+ Global Variables)

| File | Globals | Key Variables |
|------|---------|---------------|
| `Quake/WinQuake/host.c` | ~22 | `host_parms`, `host_initialized`, `host_frametime`, `realtime`, `host_framecount` |
| `Quake/WinQuake/common.c` | ~17 | `registered`, `com_modified`, `com_argc`, `com_searchpaths`, `loadsize` |
| `Quake/WinQuake/net_main.c` | ~25 | `DEFAULTnet_hostport`, `net_activeconnections`, sockets, serial config |
| `Quake/WinQuake/cl_main.c` | ~20 | `cls`, `cl`, `cl_numvisedicts`, client cvars |
| `Quake/WinQuake/sv_main.c` | ~3 | `sv`, `svs`, `fatbytes` |

**Impact**: Makes unit testing impossible, prevents multi-instance execution, creates hidden dependencies between components.

### 4.2 No Interface Definitions

The engine uses **direct function calls** between all subsystems with no interface abstractions:

- Rendering: No `renderer_interface_t` — code calls `R_RenderView()` directly
- Sound: No `audio_interface_t` — code calls `S_Update()` directly
- Input: No `input_interface_t` — code calls `IN_Move()` directly
- Network: Partial abstraction via `net_driver_t` struct, but not consistently used

### 4.3 No Configuration Management

Configuration is spread across:

| Mechanism | Location | Issues |
|-----------|----------|--------|
| Cvars (console variables) | `cvar.c` | Registered at init time, not externally configurable |
| `#define` constants | Various `.h` files | Compile-time only |
| Command-line args | `common.c` | Parsed manually with `COM_CheckParm()` |
| Config files | `config.cfg`, `default.cfg` | Proprietary format, no env var support |

---

## 5. Hard-Coded Values

### 5.1 Network Configuration

| Value | Location | Impact |
|-------|----------|--------|
| Port 26000 | `net_main.c:34` | Cannot be configured via env vars |
| `"127.0.0.1"` | `net_udp.c:341`, `net_wins.c:502` | Hard-coded loopback |
| `"localhost"` | `net_loop.c:79` | Hard-coded hostname |
| COM ports `{0x3f8, 0x2f8, 0x3e8, 0x2e8}` | `menu.c:1724`, `net_comx.c:130` | Legacy ISA hardware addresses |
| IRQ=4, BAUD=57600 | `net_main.c:71-72` | Legacy serial config |

### 5.2 Buffer Sizes and Limits

| Constant | Value | Location | Impact |
|----------|-------|----------|--------|
| `MAX_MAP_MODELS` | 256 | `bspfile.h:26` | Limits map complexity |
| `MAX_MAP_ENTITIES` | 1024 | `bspfile.h:28` | Limits entity count |
| `MAX_MAP_EDGES` | 256,000 | `bspfile.h:39` | Limits geometry |
| `MAX_MAP_SURFEDGES` | 512,000 | `bspfile.h:41` | Limits geometry |
| `MAX_MAPSTRING` | 2,048 | `client.h:97` | Limits map command strings |
| `MAX_VISEDICTS` | 256 | `client.h:306` | Limits visible entities |
| `MAX_DATAGRAM` | 1,024 | `quakedef.h:102` | Limits packet size |
| `MAX_MODELS` | 256 | `quakedef.h:109` | Limits loaded models |
| `MAX_SOUNDS` | 256 | `quakedef.h:110` | Limits loaded sounds |
| `MAXALIASVERTS` | 1,024 | `gl_model.h:317` | Limits model vertices |
| `MAXALIASFRAMES` | 256 | `gl_model.h:318` | Limits animation frames |
| `MAXALIASTRIS` | 2,048 | `gl_model.h:319` | Limits model triangles |
| `DYNAMIC_SIZE` | 0xc000 (49KB) | `zone.c:24` | Zone memory pool size |

### 5.3 File Paths

| Path | Location | Impact |
|------|----------|--------|
| `"%s/config.cfg"` | `host.c:254` | Hard-coded config path |
| `"pak%i.pak"` | `common.c:1711` | Fixed package naming |
| `"id1"` | `common.c:1775` | Default game directory |
| `"exec default.cfg"` | `menu.c:1259` | Hard-coded command |
| `"glquake/"` | `gl_mesh.c:307` | Hard-coded cache directory |

---

## 6. Build System Debt

### 6.1 Fragmented Build Files

| File | Target | Issues |
|------|--------|--------|
| `Quake/WinQuake/Makefile.linuxi386` | Linux WinQuake | GNU Make, x86 only |
| `Quake/WinQuake/Makefile.Solaris` | Solaris WinQuake | Separate from Linux |
| `Quake/QW/Makefile.Linux` | Linux QuakeWorld | Different structure |
| `Quake/QW/Makefile.Solaris` | Solaris QuakeWorld | Different structure |
| `Quake/WinQuake/WinQuake.dsp` | Windows WinQuake | MSVC 4-6 format |
| `Quake/QW/qw.dsw` | Windows QuakeWorld | MSVC workspace |

### 6.2 Build System Issues

- **No unified build**: Each platform requires different build commands
- **No dependency management**: All libraries are vendored or system-provided
- **No automated testing**: No test targets in any build file
- **No CI/CD integration**: No pipeline definitions
- **Stale project files**: MSVC `.mdp`, `.ncb`, `.opt` files are IDE state, not build instructions
- **gas2masm dependency**: Requires building a custom tool to convert assembly syntax

---

## 7. Code Duplication

### 7.1 WinQuake vs QuakeWorld

Many files are **duplicated between WinQuake and QuakeWorld** with minor variations:

| File | WinQuake | QW/client | QW/server |
|------|----------|-----------|-----------|
| `common.c` | ✅ | ✅ | ✅ |
| `cmd.c` | ✅ | ✅ | ✅ |
| `zone.c` | ✅ | ✅ | ✅ |
| `mathlib.c` | ✅ | ✅ | ✅ |
| `model.c` / `gl_model.c` | ✅ | ✅ | — |
| `pr_exec.c` / `pr_edict.c` | ✅ | — | ✅ |
| `snd_*.c` | ✅ | ✅ | — |
| Assembly files (`.s`) | 21 files | 19 files | 2 files |

This duplication means bug fixes must be applied in **multiple locations**, increasing maintenance cost.

---

## 8. Debt Priority Matrix

| Debt Item | Severity | Effort | Priority | Blocks |
|-----------|----------|--------|----------|--------|
| Unsafe string functions (664) | Critical | High | **P0** | Cloud deployment |
| Unchecked malloc (9+) | High | Low | **P0** | Cloud deployment |
| system() calls (2) | High | Low | **P0** | Cloud deployment |
| Build system fragmentation | High | Medium | **P1** | All modernization |
| Global state (105+ vars) | High | Very High | **P2** | Unit testing, multi-instance |
| Platform coupling (23 files) | High | High | **P2** | Containerization |
| Hard-coded values | Medium | Medium | **P2** | Configuration management |
| Missing abstractions | Medium | Very High | **P3** | Modularity |
| Code duplication | Medium | High | **P3** | Maintenance |
| Assembly dependency (10K+ lines) | Medium | Very High | **P4** | ARM/multi-arch support |
