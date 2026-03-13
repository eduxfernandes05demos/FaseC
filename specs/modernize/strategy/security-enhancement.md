# Security Enhancement Plan — Quake Engine

> Security modernization: replace unsafe functions, input validation, memory safety, secure networking.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This plan addresses the **664+ buffer overflow vectors**, **9+ unchecked allocations**, **2 command injection points**, and systemic lack of authentication, encryption, and input validation identified in the security audit. The plan is organized in three phases: Critical (pre-deployment), High (cloud-ready), and Hardening (production).

---

## 2. Phase 1 — Critical Security Fixes

### 2.1 Replace sprintf/vsprintf with snprintf/vsnprintf

**Scope**: 362 instances across the entire codebase

**Approach**: Mechanical replacement with buffer size enforcement.

**Pattern**:
```c
/* Before (legacy-src/desktop-engine/common.c:1286): */
sprintf(name, "%s/%s", com_gamedir, filename);

/* After: */
snprintf(name, sizeof(name), "%s/%s", com_gamedir, filename);
```

**High-Priority Files** (network-adjacent data):

| File | Instances | Risk Level |
|------|-----------|-----------|
| `legacy-src/desktop-engine/common.c` | 7+ | Critical — file path construction |
| `legacy-src/desktop-engine/net_dgrm.c` | 3+ | Critical — network address handling |
| `legacy-src/desktop-engine/console.c` | 4 | High — vsprintf with variadic args |
| `legacy-src/desktop-engine/sys_win.c` | 4 | High — error message formatting |
| `legacy-src/QW/server/sv_main.c` | 5+ | Critical — server log/data handling |
| `legacy-src/QW/client/cl_main.c` | 5+ | Critical — RCON command building |
| `legacy-src/QW/qwfwd/misc.c` | 2+ | Critical — IP formatting, user info |

**vsprintf special handling**: Replace with `vsnprintf` and add buffer size parameter:
```c
/* Before (legacy-src/desktop-engine/console.c:360): */
vsprintf(data, fmt, argptr);

/* After: */
vsnprintf(data, sizeof(data), fmt, argptr);
```

### 2.2 Replace strcpy/Q_strcpy with Bounded Copies

**Scope**: 232 instances

**Approach**: Replace with `strncpy` + explicit null termination or implement a safe `Q_strlcpy`:

```c
/* New safe copy function (replaces Q_strcpy at common.c:180): */
size_t Q_strlcpy(char *dst, const char *src, size_t dstsize) {
    size_t srclen = strlen(src);
    if (dstsize > 0) {
        size_t copylen = (srclen >= dstsize) ? dstsize - 1 : srclen;
        memcpy(dst, src, copylen);
        dst[copylen] = '\0';
    }
    return srclen;
}
```

**High-Priority Files**:

| File | Line Examples | Risk |
|------|-------------|------|
| `legacy-src/desktop-engine/common.c` | 1696, 1744, 1767, 1817 | Directory/path from user input |
| `legacy-src/desktop-engine/net_dgrm.c` | 132-133, 558, 562, 680, 683, 1074 | Network address data |
| `legacy-src/desktop-engine/gl_model.c` | 203, 1220, 1260 | Model names from file data |
| `legacy-src/QW/client/cl_demo.c` | various | User-provided demo names |

### 2.3 Replace strcat/Q_strcat with Bounded Concatenation

**Scope**: 70 instances

**Approach**: Replace with `strncat` or implement `Q_strlcat`:

```c
/* New safe concatenation (replaces Q_strcat at QW/client/common.c:218): */
size_t Q_strlcat(char *dst, const char *src, size_t dstsize) {
    size_t dstlen = strlen(dst);
    size_t srclen = strlen(src);
    if (dstlen < dstsize) {
        size_t copylen = (srclen >= dstsize - dstlen) ? dstsize - dstlen - 1 : srclen;
        memcpy(dst + dstlen, src, copylen);
        dst[dstlen + copylen] = '\0';
    }
    return dstlen + srclen;
}
```

**High-Priority Files**:

| File | Line Examples | Risk |
|------|-------------|------|
| `legacy-src/desktop-engine/host_cmd.c` | 275-278, 1054-1055 | Map/say command building |
| `legacy-src/desktop-engine/pr_cmds.c` | 41 | QuakeC string concatenation |
| `legacy-src/QW/server/sv_send.c` | 120 | Server output buffer |
| `legacy-src/QW/server/sv_main.c` | 428, 757-758, 1035 | Log/command building |
| `legacy-src/QW/client/cl_main.c` | 328-336 | RCON command building |
| `legacy-src/QW/client/keys.c` | 330 | Clipboard paste |

### 2.4 Add NULL Checks for malloc

**Scope**: 9+ instances

**Pattern**:
```c
/* Before (legacy-src/desktop-engine/host.c:791): */
com_argv = malloc(com_argc * sizeof(char *));

/* After: */
com_argv = malloc(com_argc * sizeof(char *));
if (!com_argv)
    Sys_Error("Out of memory allocating command arguments");
```

**All locations requiring fixes**:

| File | Line | Allocation |
|------|------|-----------|
| `legacy-src/desktop-engine/host.c` | 791, 796 | Startup argument parsing |
| `legacy-src/desktop-engine/gl_warp.c` | 410, 520 | Texture loading |
| `legacy-src/desktop-engine/gl_screen.c` | 618 | Screenshot buffer |
| `legacy-src/QW/client/gl_screen.c` | 651, 879 | Screenshot/resize buffers |
| `legacy-src/QW/client/cl_parse.c` | 490 | Download data buffer |
| `legacy-src/QW/client/screen.c` | 845 | Screen resize buffer |
| `legacy-src/QW/qwfwd/qwfwd.c` | 225 | Forwarding proxy allocation |

### 2.5 Remove system() Calls

**Scope**: 2 instances

| File | Line | Replacement |
|------|------|-------------|
| `legacy-src/desktop-engine/sys_linux.c` | 276 | Remove or replace with `posix_spawn()` |
| `legacy-src/QW/client/sys_linux.c` | 278 | Remove or replace with `posix_spawn()` |

---

## 3. Phase 2 — Network Security

### 3.1 Input Validation

Add validation for all data received from network:

| Data Type | Source | Validation |
|-----------|--------|-----------|
| Entity deltas | `QW/client/cl_ents.c:265` | Validate field bitmask bounds |
| Model/sound names | `cl_parse.c` | Sanitize path characters |
| Player info strings | `QW/qwfwd/misc.c:401` | Length check, character whitelist |
| Map names | `host_cmd.c:275` | Alphanumeric + underscore only |
| RCON commands | `QW/client/cl_main.c:328` | Rate limiting, input sanitization |

### 3.2 Authentication

| Component | Current | Target |
|-----------|---------|--------|
| Server join | No authentication | Token-based session auth |
| RCON | Plaintext password at `cl_main.c:330` | HMAC-authenticated commands |
| Master server | No verification | TLS + API key |

### 3.3 Encryption

| Channel | Current | Target |
|---------|---------|--------|
| Game traffic | Unencrypted UDP | DTLS 1.3 (optional) |
| RCON | Plaintext | TLS-encrypted channel |
| Health endpoint | N/A | HTTPS (mTLS for internal) |
| Streaming | N/A | WSS (WebSocket Secure) |

### 3.4 Rate Limiting

| Endpoint | Limit | Implementation |
|----------|-------|---------------|
| Connection attempts | 5/second per IP | Token bucket |
| RCON commands | 1/second | Sliding window |
| Packet processing | 100/second per client | Token bucket |
| API requests (gateway) | 100/minute per client | Rate limiter middleware |

---

## 4. Phase 3 — Container Security Hardening

### 4.1 Container Image Security

| Control | Implementation |
|---------|---------------|
| Non-root execution | `USER quake` in Dockerfile |
| Minimal base image | `ubuntu:22.04-minimal` or `distroless` |
| Read-only filesystem | `readOnlyRootFilesystem: true` in pod spec |
| No privileged access | `privileged: false`, drop all capabilities |
| Image scanning | Trivy/Snyk in CI pipeline |
| Image signing | Cosign with Sigstore |

### 4.2 Kubernetes Security

| Control | Implementation |
|---------|---------------|
| Network policies | Allow only gateway → engine communication |
| Pod security standards | Restricted profile |
| Secret management | Azure Key Vault + CSI driver |
| RBAC | Minimal service account permissions |
| Resource limits | CPU/memory limits per pod |

### 4.3 File System Security

| Control | Implementation |
|---------|---------------|
| Path sanitization | Validate all file paths against whitelist |
| Directory restriction | Chroot-like isolation to game data directory |
| Write restrictions | Read-only game data, write only to `/tmp` |
| Download validation | Checksum verification for downloaded content |

---

## 5. Security Testing

### 5.1 Static Analysis

| Tool | Target | Frequency |
|------|--------|-----------|
| CodeQL | Buffer overflows, injections | Every PR |
| Cppcheck | Null derefs, memory leaks | Every PR |
| Clang Static Analyzer | Logic errors | Every PR |

### 5.2 Dynamic Testing

| Tool | Target | Frequency |
|------|--------|-----------|
| AFL++ | Protocol fuzzing | Weekly |
| Valgrind/ASan | Memory corruption | Every PR |
| Nmap | Port scanning | Per deployment |
| OWASP ZAP | API gateway | Per deployment |

### 5.3 Compliance Validation

| Check | Tool | Threshold |
|-------|------|-----------|
| CVE scanning | Trivy | Zero critical/high |
| SBOM generation | Syft | Complete inventory |
| License compliance | Scancode | GPL v2 compatible |

---

## 6. Implementation Order

| Priority | Task | Dependencies | Effort |
|----------|------|-------------|--------|
| P0 | Replace sprintf → snprintf (362) | CMake build | 1.5 weeks |
| P0 | Replace strcpy → safe copy (232) | CMake build | 1 week |
| P0 | Replace strcat → safe concat (70) | CMake build | 0.5 weeks |
| P0 | Add malloc NULL checks (9) | None | 1 day |
| P0 | Remove system() calls (2) | None | 1 day |
| P1 | Network input validation | Security fixes | 1 week |
| P1 | RCON authentication | Network fixes | 1 week |
| P2 | TLS/DTLS encryption | Auth | 2 weeks |
| P2 | Rate limiting | Network | 1 week |
| P3 | Container hardening | Dockerfile | 1 week |
| P3 | Kubernetes security | AKS deployment | 1 week |
