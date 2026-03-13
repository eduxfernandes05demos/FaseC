# Task: Security Remediation — Buffer Overflows

> Replace all `sprintf`/`strcpy`/`strcat` with safe alternatives, add bounds checking.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P0 — Critical |
| **Estimated Complexity** | High |
| **Estimated Effort** | 3 weeks |
| **Dependencies** | CMake build system (task-build-system) |
| **Blocks** | All network-facing deployment, containerization |
| **Phase** | Phase 1 — Security Hardening |

---

## Objective

Eliminate all buffer overflow vulnerabilities by replacing unbounded string operations with bounds-checked alternatives throughout the entire Quake codebase.

---

## Scope

| Category | Count | Replacement |
|----------|-------|-------------|
| `sprintf` / `vsprintf` | 362 instances | `snprintf` / `vsnprintf` |
| `strcpy` / `Q_strcpy` | 232 instances | `Q_strlcpy` (new safe function) |
| `strcat` / `Q_strcat` | 70 instances | `Q_strlcat` (new safe function) |
| Unchecked `malloc` | 9+ instances | Add NULL check + `Sys_Error` |
| `system()` calls | 2 instances | Remove or replace with `posix_spawn()` |

**Total**: 664+ code changes across 240 `.c` files

---

## Implementation Steps

### Step 1: Implement Safe String Functions

Create new utility functions in `legacy-src/desktop-engine/common.c` (or new `safe_str.c`):

```c
/* Replace Q_strcpy at common.c:180 */
size_t Q_strlcpy(char *dst, const char *src, size_t dstsize);

/* Replace Q_strcat at QW/client/common.c:218 */
size_t Q_strlcat(char *dst, const char *src, size_t dstsize);
```

### Step 2: Replace sprintf (362 instances)

**Priority files** (network-adjacent, process first):

| File | Line Examples | Instance Count |
|------|-------------|----------------|
| `legacy-src/desktop-engine/common.c` | 1286, 1424, 1437, 1439, 1441, 1711 | 7+ |
| `legacy-src/desktop-engine/net_dgrm.c` | 134 | 3+ |
| `legacy-src/desktop-engine/console.c` | 360, 384, 433, 454 | 4 (vsprintf) |
| `legacy-src/desktop-engine/sys_win.c` | 367, 373, 376, 435 | 4 |
| `legacy-src/desktop-engine/gl_model.c` | 1217, 1441, 1466, 1686 | 4 |
| `legacy-src/desktop-engine/vid_ext.c` | 341, 347, 356, 362 | 4 |
| `legacy-src/QW/server/sv_main.c` | various | 5+ |
| `legacy-src/QW/client/cl_main.c` | various | 5+ |
| `legacy-src/QW/qwfwd/misc.c` | 33, 401 | 2 |

**Pattern**:
```c
/* Before: */
sprintf(name, "%s/%s", com_gamedir, filename);
/* After: */
snprintf(name, sizeof(name), "%s/%s", com_gamedir, filename);
```

### Step 3: Replace strcpy (232 instances)

**Priority files**:

| File | Line Examples |
|------|-------------|
| `legacy-src/desktop-engine/common.c` | 222, 869, 1432, 1448, 1665, 1671, 1696, 1702, 1744, 1746, 1767, 1770, 1817 |
| `legacy-src/desktop-engine/net_dgrm.c` | 132, 133, 558, 562, 680, 683, 1074 |
| `legacy-src/desktop-engine/gl_model.c` | 203, 1220, 1260 |
| `legacy-src/QW/client/cl_demo.c` | various |

**Pattern**:
```c
/* Before: */
strcpy(com_gamedir, dir);
/* After: */
Q_strlcpy(com_gamedir, dir, sizeof(com_gamedir));
```

### Step 4: Replace strcat (70 instances)

**Priority files**:

| File | Line Examples |
|------|-------------|
| `legacy-src/desktop-engine/host_cmd.c` | 275, 276, 278, 296, 297, 1054, 1055, 1102, 1118, 1119 |
| `legacy-src/desktop-engine/cmd.c` | 242, 244, 263, 264, 386, 388, 390 |
| `legacy-src/desktop-engine/pr_cmds.c` | 41 |
| `legacy-src/QW/server/sv_send.c` | 120 |
| `legacy-src/QW/server/sv_main.c` | 428, 757, 758, 1035 |
| `legacy-src/QW/client/cl_main.c` | 328, 330, 331, 335, 336 |
| `legacy-src/QW/client/keys.c` | 330, 567, 569 |

**Pattern**:
```c
/* Before: */
strcat(cls.mapstring, Cmd_Argv(i));
/* After: */
Q_strlcat(cls.mapstring, Cmd_Argv(i), sizeof(cls.mapstring));
```

### Step 5: Add malloc NULL Checks (9+ instances)

| File | Line | Fix |
|------|------|-----|
| `legacy-src/desktop-engine/host.c` | 791 | Add `if (!com_argv) Sys_Error("Out of memory");` |
| `legacy-src/desktop-engine/host.c` | 796 | Add `if (!p) Sys_Error("Out of memory");` |
| `legacy-src/desktop-engine/gl_warp.c` | 410 | Add NULL check after `malloc(count * 4)` |
| `legacy-src/desktop-engine/gl_warp.c` | 520 | Add NULL check after `malloc(numPixels*4)` |
| `legacy-src/desktop-engine/gl_screen.c` | 618 | Add NULL check after `malloc(...)` |
| `legacy-src/QW/client/gl_screen.c` | 651 | Add NULL check after `malloc(...)` |
| `legacy-src/QW/client/gl_screen.c` | 879 | Add NULL check after `malloc(...)` |
| `legacy-src/QW/client/cl_parse.c` | 490 | Add NULL check after `malloc(size)` |
| `legacy-src/QW/client/screen.c` | 845 | Add NULL check after `malloc(w*h)` |
| `legacy-src/QW/qwfwd/qwfwd.c` | 225 | Add NULL check after `malloc(sizeof *p)` |

### Step 6: Remove system() Calls

| File | Line | Action |
|------|------|--------|
| `legacy-src/desktop-engine/sys_linux.c` | 276 | Remove or replace with safe alternative |
| `legacy-src/QW/client/sys_linux.c` | 278 | Remove or replace with safe alternative |

---

## Testing Requirements

| Test | Description | Framework |
|------|-------------|-----------|
| `test_strlcpy_basic` | Verify Q_strlcpy copies correctly | Unity |
| `test_strlcpy_truncation` | Verify truncation when dst too small | Unity |
| `test_strlcpy_empty` | Handle empty strings | Unity |
| `test_strlcat_basic` | Verify Q_strlcat appends correctly | Unity |
| `test_strlcat_overflow` | Verify overflow prevention | Unity |
| `test_snprintf_paths` | Verify path construction doesn't truncate | Unity |
| `test_malloc_null_handling` | Verify graceful handling of OOM | Unity |
| `test_no_unsafe_functions` | Grep-based test that no unsafe functions remain | Shell script |

---

## Acceptance Criteria

- [ ] `grep -rn "\bsprintf\b" legacy-src/ --include="*.c" | grep -v snprintf | wc -l` returns 0
- [ ] `grep -rn "\bvsprintf\b" legacy-src/ --include="*.c" | grep -v vsnprintf | wc -l` returns 0
- [ ] `grep -rn "\bstrcpy\b" legacy-src/ --include="*.c" | grep -v strncpy | grep -v strlcpy | wc -l` returns 0 (excluding safe wrappers)
- [ ] `grep -rn "\bstrcat\b" legacy-src/ --include="*.c" | grep -v strncat | grep -v strlcat | wc -l` returns 0 (excluding safe wrappers)
- [ ] `grep -rn "\bsystem(" legacy-src/ --include="*.c" | wc -l` returns 0
- [ ] All `malloc` calls have NULL checks (verified by static analysis)
- [ ] Engine compiles without warnings
- [ ] Timedemo completes without crashes
- [ ] Network play works (QW client connects to QW server)
- [ ] AddressSanitizer build runs timedemo without findings
- [ ] All new unit tests pass
