# Security Audit — Quake Engine

> Security vulnerability assessment for modernization planning.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

The Quake engine contains **critical security vulnerabilities** inherent to 1996-era C development practices. The codebase was designed for trusted LAN environments and lacks the defensive programming practices required for internet-facing or cloud-deployed services. This audit identifies **664+ buffer overflow vectors**, **9+ unchecked allocations**, **2 command injection points**, and multiple categories of input validation failures.

**Risk Rating**: **CRITICAL** — The engine MUST NOT be deployed in any network-facing configuration without comprehensive security remediation.

---

## 2. Buffer Overflow Vulnerabilities

### 2.1 sprintf / vsprintf (362 instances)

**Severity**: Critical — Remote Code Execution (RCE)

Unbounded `sprintf` writes formatted data to fixed-size stack buffers without length checking. An attacker controlling format arguments can overwrite the stack, enabling arbitrary code execution.

**Representative Vulnerable Patterns**:

| File | Line | Code | Buffer Risk |
|------|------|------|-------------|
| `Quake/WinQuake/common.c` | 1286 | `sprintf(name, "%s/%s", com_gamedir, filename);` | Path construction → stack overflow |
| `Quake/WinQuake/common.c` | 1424 | `sprintf(netpath, "%s/%s", search->filename, filename);` | Path construction → stack overflow |
| `Quake/WinQuake/console.c` | 360 | `vsprintf(data, fmt, argptr);` | Variadic format → stack overflow |
| `Quake/WinQuake/sys_win.c` | 367 | `vsprintf(text, error, argptr);` | Error message → stack overflow |
| `Quake/WinQuake/sys_win.c` | 376 | `sprintf(text2, "ERROR: %s\n", text);` | Cascading overflow |
| `Quake/WinQuake/gl_model.c` | 1217 | `sprintf(name, "*%i", i+1);` | Model name → limited risk |
| `Quake/WinQuake/net_dgrm.c` | 134 | Format IP addresses into fixed buffers | Network data → stack overflow |
| `Quake/WinQuake/vid_ext.c` | 341-362 | Mode name formatting into fixed arrays | Display mode → stack overflow |
| `Quake/QW/qwfwd/misc.c` | 33 | `sprintf(s, "%i.%i.%i.%i:%i", ...)` | IP formatting → overflow |
| `Quake/QW/qwfwd/misc.c` | 401 | `sprintf(new, "\\%s\\%s", key, value);` | User info → overflow |

**Highest-risk files** (most sprintf instances with network-adjacent data):
- `Quake/WinQuake/common.c` — 7+ instances
- `Quake/WinQuake/net_dgrm.c` — 3+ instances
- `Quake/QW/server/sv_main.c` — 5+ instances
- `Quake/QW/client/cl_main.c` — 5+ instances

### 2.2 strcpy / Q_strcpy (232 instances)

**Severity**: Critical — Remote Code Execution (RCE)

Unbounded `strcpy` copies source strings to destination buffers without length validation. The engine defines its own `Q_strcpy` at `Quake/WinQuake/common.c:180` which is equally unsafe.

**High-Risk Examples**:

| File | Line | Code | Risk |
|------|------|------|------|
| `Quake/WinQuake/common.c` | 1696 | `strcpy(com_gamedir, dir);` | Directory path from user config |
| `Quake/WinQuake/common.c` | 1744 | `strcpy(basedir, com_argv[i+1]);` | Command-line arg → overflow |
| `Quake/WinQuake/common.c` | 1767 | `strcpy(com_cachedir, com_argv[i+1]);` | Command-line arg → overflow |
| `Quake/WinQuake/net_dgrm.c` | 132 | `Q_strcpy(addrStr, inet_ntoa(...));` | Network address → overflow |
| `Quake/WinQuake/net_dgrm.c` | 558-683 | Multiple Q_strcpy with network data | Network → overflow |
| `Quake/WinQuake/gl_model.c` | 203 | `strcpy(mod->name, name);` | Model name → overflow |
| `Quake/QW/client/cl_demo.c` | various | `strcpy(name, Cmd_Argv(1));` | User input → overflow |
| `Quake/QW/client/net_wins.c` | various | `strcpy(copy, s);` | Network data → overflow |

### 2.3 strcat / Q_strcat (70 instances)

**Severity**: High — Denial of Service / Code Execution

Unbounded `strcat` appends strings without checking available buffer space.

**High-Risk Examples**:

| File | Line | Code | Risk |
|------|------|------|------|
| `Quake/WinQuake/host_cmd.c` | 275-278 | Map string concatenation | User command → overflow |
| `Quake/WinQuake/host_cmd.c` | 1054-1055 | Say command text building | Chat message → overflow |
| `Quake/WinQuake/pr_cmds.c` | 41 | `strcat(out, G_STRING(...));` | QuakeC string → overflow |
| `Quake/QW/server/sv_send.c` | 120 | `strcat(outputbuf, msg);` | Server output → overflow |
| `Quake/QW/server/sv_main.c` | 428 | `strcat(data, svs.log_buf[...]);` | Log data → overflow |
| `Quake/QW/client/cl_main.c` | 328-336 | RCON command building | Auth data → overflow |
| `Quake/QW/client/keys.c` | 330 | `strcat(key_lines[edit_line], textCopied);` | Clipboard paste → overflow |

---

## 3. Memory Safety Vulnerabilities

### 3.1 Unchecked Allocations (9+ instances)

**Severity**: High — Denial of Service / Null Pointer Dereference

| File | Line | Allocation | Risk |
|------|------|-----------|------|
| `Quake/WinQuake/host.c` | 791 | `malloc(com_argc * sizeof(char *))` | Startup crash on OOM |
| `Quake/WinQuake/host.c` | 796 | `malloc(len)` | Startup crash on OOM |
| `Quake/WinQuake/gl_warp.c` | 410 | `malloc(count * 4)` | Texture crash on OOM |
| `Quake/WinQuake/gl_warp.c` | 520 | `malloc(numPixels*4)` | TGA loading crash on OOM |
| `Quake/WinQuake/gl_screen.c` | 618 | `malloc(glwidth*glheight*3 + 18)` | Screenshot crash on OOM |
| `Quake/QW/client/cl_parse.c` | 490 | `malloc(size)` | Download crash on OOM |
| `Quake/QW/qwfwd/qwfwd.c` | 225 | `malloc(sizeof *p)` | Forwarding crash on OOM |

### 3.2 Custom Memory Allocator Risks

The Zone allocator (`Quake/WinQuake/zone.c`) implements its own heap management:

- **Line 155**: `Z_TagMalloc()` — Walks a free list; no bounds checking on the arena
- **Line 99**: `Z_Free()` — Validates block sentinels but uses assertions, not graceful error handling
- **Line 247**: `Z_CheckHeap()` — Debug-only heap validation

**Risks**:
- Zone corruption can overwrite adjacent memory blocks silently
- Hunk allocator has no overflow protection — returns NULL only when fully exhausted
- Cache allocator can evict data needed by other subsystems under memory pressure

---

## 4. Command Injection

### 4.1 system() Calls

**Severity**: High — Arbitrary Command Execution

| File | Line | Code | Context |
|------|------|------|---------|
| `Quake/WinQuake/sys_linux.c` | 276 | `system(cmd);` | Executing shell commands |
| `Quake/QW/client/sys_linux.c` | 278 | `system(cmd);` | Executing shell commands |

If `cmd` contains any user-controlled content without proper sanitization, an attacker could execute arbitrary shell commands on the host system.

---

## 5. Network Security

### 5.1 No Authentication

- **No server authentication**: Clients connect to any server without verification
- **No client authentication**: Servers accept any connecting client
- **RCON (remote console)**: Password sent in plaintext (`Quake/QW/client/cl_main.c:330`)
- **No TLS/encryption**: All network traffic is unencrypted

### 5.2 No Input Validation on Network Data

- Packet parsing trusts data from network without validation (`Quake/WinQuake/net_dgrm.c:161`)
- Entity delta parsing trusts field bitmasks (`Quake/QW/client/cl_ents.c:265`)
- Model/texture names from network are used directly in file operations
- String data from network is copied into fixed buffers without length checks

### 5.3 No Rate Limiting

- No connection rate limiting — denial of service via connection flooding
- No packet rate limiting — network amplification attacks possible
- No bandwidth throttling — resource exhaustion attacks

### 5.4 Hard-Coded Network Configuration

| Value | Location | Risk |
|-------|----------|------|
| Port 26000 | `net_main.c:34` | Predictable, well-known port |
| `"127.0.0.1"` | `net_udp.c:341`, `net_wins.c:502` | SSRF potential if misused |

---

## 6. File System Security

### 6.1 Path Traversal

File paths are constructed using `sprintf` with user-controlled directory and file names:

| File | Line | Risk |
|------|------|------|
| `Quake/WinQuake/common.c` | 1286 | `sprintf(name, "%s/%s", com_gamedir, filename)` — No path sanitization |
| `Quake/WinQuake/common.c` | 1424 | `sprintf(netpath, "%s/%s", search->filename, filename)` — Path traversal |
| `Quake/QW/client/cl_parse.c` | 189 | Download temp file naming — Path traversal via server |

### 6.2 No File Permission Checks

- Files are opened with `fopen()` without permission validation
- No chroot or sandboxing of file access
- Game data directory is fully writable by the engine process

---

## 7. Vulnerability Summary

| Category | Count | Severity | Exploitability |
|----------|-------|----------|----------------|
| Buffer overflows (sprintf) | 362 | Critical | Remote (via network data) |
| Buffer overflows (strcpy) | 232 | Critical | Remote (via network data) |
| Buffer overflows (strcat) | 70 | High | Remote (via network data) |
| Unchecked allocations | 9+ | High | Local (OOM condition) |
| Command injection | 2 | High | Local (user input) |
| No authentication | System-wide | Critical | Remote |
| No encryption | System-wide | High | Network (passive sniffing) |
| No input validation | System-wide | Critical | Remote |
| Path traversal | 3+ | High | Remote (via server) |
| No rate limiting | System-wide | Medium | Remote (DoS) |

---

## 8. Remediation Priority

### Phase 1 — Critical (Before Any Network Deployment)

1. Replace all `sprintf`/`vsprintf` with `snprintf`/`vsnprintf`
2. Replace all `strcpy`/`Q_strcpy` with bounds-checked alternatives
3. Replace all `strcat`/`Q_strcat` with bounds-checked alternatives
4. Add NULL checks after all `malloc` calls
5. Remove or sandbox `system()` calls

### Phase 2 — High (Before Cloud Deployment)

6. Implement network input validation
7. Add authentication for server connections
8. Encrypt RCON and sensitive network traffic
9. Add path sanitization for file operations
10. Implement rate limiting

### Phase 3 — Medium (Cloud Hardening)

11. Implement TLS for all network communications
12. Add connection throttling and DDoS protection
13. Implement file system sandboxing
14. Add security logging and audit trails
15. Container security hardening (non-root, read-only filesystem, capability dropping)
