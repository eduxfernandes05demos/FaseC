# Technology Upgrade Plan — Quake Engine

> Technology modernization plan: compiler upgrades, safe C patterns, modern build system, cross-platform abstractions.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document details the technology upgrade path from the 1996 C89 toolchain to a modern C11/C17 development environment with CMake, static analysis, sanitizers, and cross-platform abstractions. The upgrade preserves the C language (no C++ migration) while adopting modern safety features and tooling.

---

## 2. Compiler Upgrade

### 2.1 Current State

| Component | Current | Notes |
|-----------|---------|-------|
| C Standard | C89 (ANSI C) | No `inline`, no `_Bool`, no `restrict`, no VLAs |
| Compiler (Linux) | GCC (unspecified version) | `Quake/WinQuake/Makefile.linuxi386` |
| Compiler (Windows) | MSVC 4-6 | `Quake/WinQuake/WinQuake.dsp` |
| Compiler (Solaris) | Sun cc | `Quake/WinQuake/Makefile.Solaris` |

### 2.2 Target State

| Component | Target | Rationale |
|-----------|--------|-----------|
| C Standard | **C11** (with C17 compatibility) | `_Static_assert`, `_Alignof`, `_Noreturn`, `<stdatomic.h>` |
| Primary Compiler | **GCC 12+** / **Clang 15+** | Modern warnings, sanitizers, static analysis |
| Windows Compiler | **MSVC 2022** or **Clang-CL** | C11 support, compatible with CMake |
| Warning Level | `-Wall -Wextra -Werror` | Catch issues at compile time |

### 2.3 Compiler Features to Adopt

| Feature | C Standard | Usage |
|---------|-----------|-------|
| `_Static_assert` | C11 | Compile-time buffer size validation |
| `_Noreturn` | C11 | Mark `Sys_Error()`, `Sys_Quit()` |
| `restrict` | C99 | Optimize pointer-heavy rendering code |
| `inline` | C99 | Replace macro functions |
| `<stdbool.h>` | C99 | Replace `int` flags with `bool` |
| `<stdint.h>` | C99 | Replace `int` with `int32_t`, `uint8_t` |

---

## 3. Build System Migration

### 3.1 Current Build Files (to be replaced)

| File | Target |
|------|--------|
| `Quake/WinQuake/Makefile.linuxi386` | Linux WinQuake |
| `Quake/WinQuake/Makefile.Solaris` | Solaris WinQuake |
| `Quake/QW/Makefile.Linux` | Linux QuakeWorld |
| `Quake/QW/Makefile.Solaris` | Solaris QuakeWorld |
| `Quake/WinQuake/WinQuake.dsp` | MSVC WinQuake |
| `Quake/WinQuake/WinQuake.dsw` | MSVC Workspace |
| `Quake/QW/qw.dsw` | MSVC QW Workspace |

### 3.2 CMake Structure

```
Quake/
├── CMakeLists.txt              # Top-level project
├── cmake/
│   ├── CompilerOptions.cmake   # Compiler flags, sanitizers
│   ├── PlatformDetect.cmake    # OS/arch detection
│   └── FindDependencies.cmake  # External library discovery
├── WinQuake/
│   ├── CMakeLists.txt          # WinQuake targets
│   └── ...
├── QW/
│   ├── CMakeLists.txt          # QuakeWorld targets
│   ├── client/
│   │   └── CMakeLists.txt      # QW client target
│   └── server/
│       └── CMakeLists.txt      # QW dedicated server target
└── tests/
    └── CMakeLists.txt          # Test targets
```

### 3.3 CMake Features

| Feature | Purpose |
|---------|---------|
| `target_compile_features(... c_std_11)` | Enforce C11 standard |
| `CMAKE_EXPORT_COMPILE_COMMANDS ON` | IDE/tool integration |
| `add_sanitizers()` | ASan, UBSan, TSan support |
| `CPack` | Package generation (DEB, RPM, ZIP) |
| `CTest` | Test framework integration |
| `FetchContent` | Dependency management |
| Cross-compilation toolchain files | ARM, multi-arch support |

---

## 4. Safe C Patterns

### 4.1 String Safety

| Unsafe | Replacement | Standard |
|--------|-------------|----------|
| `sprintf(buf, fmt, ...)` | `snprintf(buf, sizeof(buf), fmt, ...)` | C99 |
| `vsprintf(buf, fmt, va)` | `vsnprintf(buf, sizeof(buf), fmt, va)` | C99 |
| `strcpy(dst, src)` | `strncpy(dst, src, sizeof(dst)); dst[sizeof(dst)-1]='\0';` | C89 |
| `strcat(dst, src)` | `strncat(dst, src, sizeof(dst)-strlen(dst)-1)` | C89 |
| `Q_strcpy()` at `common.c:180` | Remove; use `strncpy` directly | — |
| `Q_strcat()` at `QW/client/common.c:218` | Remove; use `strncat` directly | — |

### 4.2 Memory Safety

| Unsafe | Replacement |
|--------|-------------|
| `malloc(size)` without NULL check | `safe_malloc(size)` wrapper with NULL check + error handling |
| `system(cmd)` | Remove; use `posix_spawn()` or dedicated subprocess API |

### 4.3 Integer Safety

| Pattern | Risk | Mitigation |
|---------|------|-----------|
| `int` for sizes | Overflow on 64-bit | Use `size_t` for sizes, `ptrdiff_t` for offsets |
| Unchecked arithmetic | Integer overflow | Add overflow checks for allocation sizes |
| Pointer truncation | 64-bit incompatibility | Use `intptr_t`/`uintptr_t` for pointer arithmetic |

---

## 5. Static Analysis Integration

### 5.1 Tools

| Tool | Purpose | Integration |
|------|---------|-------------|
| **Clang Static Analyzer** | Bug detection | `scan-build cmake --build .` |
| **Cppcheck** | C code analysis | `cppcheck --enable=all --std=c11` |
| **CodeQL** | Security vulnerability detection | GitHub Actions integration |
| **Clang-Tidy** | Code modernization | `.clang-tidy` configuration |
| **Valgrind** | Runtime memory analysis | CTest integration |

### 5.2 Sanitizers

| Sanitizer | Flag | Purpose |
|-----------|------|---------|
| AddressSanitizer | `-fsanitize=address` | Buffer overflows, use-after-free |
| UndefinedBehaviorSanitizer | `-fsanitize=undefined` | Integer overflow, null deref |
| MemorySanitizer | `-fsanitize=memory` | Uninitialized memory reads |
| ThreadSanitizer | `-fsanitize=thread` | Data races (for future multi-threading) |

---

## 6. Cross-Platform Abstraction

### 6.1 Platform Abstraction Layer

Replace compile-time platform selection with runtime-selectable backends:

```c
/* New: platform_interface.h */
typedef struct {
    int  (*init)(void);
    void (*shutdown)(void);
    double (*get_time)(void);
    void (*error)(const char *msg);
    void (*quit)(void);
    int  (*file_exists)(const char *path);
} platform_interface_t;

/* Implementations */
extern platform_interface_t platform_linux;    /* sys_linux.c */
extern platform_interface_t platform_win32;    /* sys_win.c */
extern platform_interface_t platform_headless; /* sys_headless.c (new) */
```

### 6.2 Files to Abstract

| Current Files | New Interface | Backend Files |
|---------------|---------------|---------------|
| `sys_win.c`, `sys_linux.c`, `sys_dos.c` | `platform_interface_t` | Keep + add `sys_headless.c` |
| `vid_win.c`, `vid_svgalib.c`, `vid_x.c` | `video_interface_t` | Keep + add `vid_headless.c` |
| `snd_win.c`, `snd_linux.c`, `snd_dos.c` | `audio_interface_t` | Keep + enhance `snd_null.c` |
| `in_win.c`, `in_dos.c`, `in_sun.c` | `input_interface_t` | Keep + enhance `in_null.c` |
| `cd_win.c`, `cd_linux.c` | `cdaudio_interface_t` | Keep + enhance `cd_null.c` |

---

## 7. Assembly Replacement Strategy

### 7.1 Phased Approach

| Phase | Action | Files | Risk |
|-------|--------|-------|------|
| 1 | Add C fallbacks controlled by `#ifndef USE_ASM` | All 21 `.s` files | Low |
| 2 | Validate C fallbacks match assembly output | Test suite | Medium |
| 3 | Add SIMD intrinsics (SSE2/NEON) for critical paths | `d_draw.s`, `d_polysa.s`, `r_edgea.s` | Medium |
| 4 | Default to C with optional assembly | CMake option | Low |
| 5 | Deprecate assembly files | Build system | Low |

### 7.2 Priority Assembly Files

| File | Lines | C Fallback Complexity |
|------|-------|-----------------------|
| `d_polysa.s` | 1,744 | High — complex polygon rasterization |
| `d_draw.s` | 1,037 | High — pixel drawing inner loops |
| `r_edgea.s` | 750 | Medium — edge processing |
| `snd_mixa.s` | 218 | Low — straightforward mixing loop |
| `math.s` | 418 | Low — standard math operations |

---

## 8. Dependency Management

### 8.1 Current Dependencies

| Dependency | Source | Management |
|------------|--------|-----------|
| DirectX SDK | `Quake/WinQuake/dxsdk/` | Vendored (outdated) |
| Scitech MGL | `Quake/WinQuake/scitech/` | Vendored (obsolete) |
| OpenGL | System | Implicit link |
| POSIX/Win32 | System | Implicit |

### 8.2 Target Dependencies

| Dependency | Purpose | Management |
|------------|---------|-----------|
| SDL2 | Cross-platform video/audio/input | CMake `FetchContent` or system package |
| OpenGL 3.3+ | Modern rendering (optional) | System |
| libcurl | HTTP health endpoints | CMake `FetchContent` |
| cJSON | Structured logging | CMake `FetchContent` |
| Unity (test) | Unit testing framework | CMake `FetchContent` |
| FFmpeg/libx264 | Video encoding for streaming | System package |

---

## 9. Implementation Order

```
1. CMake build system             ─→  Enables all other work
2. Compiler upgrade (C11)         ─→  Enables modern patterns
3. Static analysis integration    ─→  Catches issues early
4. Safe string replacements       ─→  Security prerequisite
5. Memory safety fixes            ─→  Security prerequisite
6. Platform abstraction layer     ─→  Enables headless mode
7. Cross-platform testing (CI)    ─→  Validates all changes
8. Assembly C fallbacks           ─→  Enables multi-arch
9. SDL2 integration (optional)    ─→  Modern platform layer
10. SIMD intrinsics               ─→  Performance parity
```
