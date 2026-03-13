# Task: Dependency Upgrade — Build System Migration to CMake

> Migrate from Makefiles/MSVC .mak to CMake, add cross-compilation support.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P0 — Foundation |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 2-3 weeks |
| **Dependencies** | None (first task) |
| **Blocks** | All subsequent modernization tasks |
| **Phase** | Phase 0 — Foundation |

---

## Objective

Replace all platform-specific build files with a unified CMake build system that supports Linux, cross-compilation, assembly toggling, and multiple build targets (WinQuake, QW client, QW server).

---

## Current Build Files (to be replaced)

| File | Purpose | Targets |
|------|---------|---------|
| `legacy-src/desktop-engine/Makefile.linuxi386` | Linux WinQuake build | `squake`, `glquake`, `unixded` |
| `legacy-src/desktop-engine/Makefile.Solaris` | Solaris WinQuake build | Solaris targets |
| `legacy-src/QW/Makefile.Linux` | QuakeWorld Linux build | `qwcl`, `glqwcl`, `qwsv` |
| `legacy-src/QW/Makefile.Solaris` | QuakeWorld Solaris build | Solaris targets |
| `legacy-src/desktop-engine/WinQuake.dsp` | MSVC 4-6 project | WinQuake Windows |
| `legacy-src/desktop-engine/WinQuake.dsw` | MSVC workspace | WinQuake workspace |
| `legacy-src/desktop-engine/WinQuake.mdp` | MSVC makefile project | WinQuake |
| `legacy-src/QW/qw.dsw` | MSVC workspace | QuakeWorld workspace |

---

## Implementation Steps

### Step 1: Create Root CMakeLists.txt

**New file**: `legacy-src/CMakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.16)
project(Quake C)

# Options
option(USE_ASM "Use x86 assembly optimizations" OFF)
option(BUILD_GLQUAKE "Build OpenGL renderer" ON)
option(BUILD_DEDICATED "Build dedicated server" ON)
option(HEADLESS_MODE "Build with headless mode support" OFF)

# C standard
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Compiler warnings
add_compile_options(-Wall -Wextra)

# Platform detection
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|i[36]86")
    set(IS_X86 TRUE)
endif()

# Subdirectories
add_subdirectory(WinQuake)
add_subdirectory(QW)
```

### Step 2: Create WinQuake CMakeLists.txt

**New file**: `legacy-src/desktop-engine/CMakeLists.txt`

Extract source file lists from `Makefile.linuxi386`:

**Software renderer sources** (from Makefile target `squake`):
- Core: `chase.c`, `cl_demo.c`, `cl_input.c`, `cl_main.c`, `cl_parse.c`, `cl_tent.c`, `cmd.c`, `common.c`, `console.c`, `crc.c`, `cvar.c`, `host.c`, `host_cmd.c`, `keys.c`, `mathlib.c`, `menu.c`, `net_loop.c`, `net_main.c`, `net_none.c`, `net_udp.c`, `net_bsd.c`, `net_dgrm.c`, `nonintel.c`, `pr_cmds.c`, `pr_edict.c`, `pr_exec.c`, `sbar.c`, `screen.c`, `snd_dma.c`, `snd_mem.c`, `snd_mix.c`, `sv_main.c`, `sv_move.c`, `sv_phys.c`, `sv_user.c`, `view.c`, `wad.c`, `world.c`, `zone.c`
- Rendering: `d_edge.c`, `d_fill.c`, `d_init.c`, `d_modech.c`, `d_part.c`, `d_polyse.c`, `d_scan.c`, `d_sky.c`, `d_sprite.c`, `d_surf.c`, `d_vars.c`, `d_zpoint.c`, `draw.c`, `model.c`, `r_aclip.c`, `r_alias.c`, `r_bsp.c`, `r_draw.c`, `r_edge.c`, `r_efrag.c`, `r_light.c`, `r_main.c`, `r_misc.c`, `r_part.c`, `r_sky.c`, `r_sprite.c`, `r_surf.c`, `r_vars.c`
- Platform: `sys_linux.c`, `vid_svgalib.c` (or `vid_x.c`), `snd_linux.c`, `cd_linux.c`

**Assembly sources** (conditionally included when `USE_ASM=ON` and `IS_X86`):
- `d_copy.s`, `d_draw.s`, `d_draw16.s`, `d_parta.s`, `d_polysa.s`, `d_scana.s`, `d_spr8.s`, `d_varsa.s`, `math.s`, `r_aclipa.s`, `r_aliasa.s`, `r_drawa.s`, `r_edgea.s`, `r_varsa.s`, `snd_mixa.s`, `surf8.s`, `surf16.s`, `worlda.s`

### Step 3: Create QW CMakeLists.txt

**New file**: `legacy-src/QW/CMakeLists.txt`

Extract source lists from `Makefile.Linux` for:
- `qwcl` (QuakeWorld client) — `legacy-src/QW/client/*.c`
- `qwsv` (QuakeWorld dedicated server) — `legacy-src/QW/server/*.c`

### Step 4: Add CTest Integration

```cmake
# In root CMakeLists.txt
enable_testing()
add_subdirectory(tests)
```

**New file**: `legacy-src/tests/CMakeLists.txt` — Test target setup

### Step 5: Add gas2masm Build (Windows assembly support)

```cmake
# Build the gas2masm converter tool
add_executable(gas2masm QW/gas2masm/gas2masm.c)
```

Source: `legacy-src/QW/gas2masm/gas2masm.c`

### Step 6: Preserve Original Build Files

Move to `legacy-src/legacy-build/`:
```bash
mkdir -p legacy-src/legacy-build
mv legacy-src/desktop-engine/Makefile.* legacy-src/legacy-build/
mv legacy-src/desktop-engine/WinQuake.dsp legacy-src/legacy-build/
# ... etc
```

---

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| CMake 3.16 minimum | Widely available, supports `FetchContent` |
| C11 standard | Modern features while maintaining C compatibility |
| Assembly off by default | Ensures portability; opt-in for x86 performance |
| Separate targets per binary | Clean dependency management |
| CTest integration | Standard test runner for CI |

---

## Compiler Flags (from Makefile.linuxi386)

Extract and translate key flags:

| Makefile Flag | CMake Equivalent |
|--------------|------------------|
| `-O2` | `CMAKE_C_FLAGS_RELEASE` |
| `-ffast-math` | `target_compile_options(... -ffast-math)` |
| `-Did386` | `target_compile_definitions(... id386)` (when USE_ASM) |
| `-DELF` | `target_compile_definitions(... ELF)` |
| `-DQUAKE_GAME` | `target_compile_definitions(... QUAKE_GAME)` |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_cmake_build` | CMake configures and builds without errors |
| `test_binary_runs` | Built binary starts and exits cleanly with `-version` |
| `test_asm_toggle` | Build succeeds with `USE_ASM=ON` and `USE_ASM=OFF` |
| `test_dedicated_server` | Dedicated server target builds and runs |
| `test_gl_target` | OpenGL target builds (if OpenGL found) |

---

## Acceptance Criteria

- [ ] `cmake -B build && cmake --build build` succeeds on Ubuntu 22.04
- [ ] WinQuake software renderer binary built (`squake`)
- [ ] QW dedicated server binary built (`qwsv`)
- [ ] QW client binary built (`qwcl`)
- [ ] Build with `USE_ASM=OFF` succeeds (C-only build)
- [ ] Build with `USE_ASM=ON` succeeds on x86 (assembly included)
- [ ] Original Makefiles preserved in `legacy-build/`
- [ ] CTest framework integrated (even if no tests yet)
- [ ] GitHub Actions workflow uses CMake for builds
- [ ] Build time ≤ 120% of original Makefile build time
