# Mitigation Strategies — Quake Engine Modernization

> Risk mitigation approaches: incremental migration, feature flags, canary deployments.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document defines mitigation strategies for each identified risk, providing actionable approaches to reduce likelihood and impact. Strategies are organized by risk category and mapped to specific implementation techniques.

---

## 2. Technical Risk Mitigations

### M-T1: Security Fixes — Incremental Validation

**Risk**: Security remediation introduces bugs (replacing 664 string operations)

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **File-by-file commits** | One commit per file, each individually tested | Reduces blast radius |
| **Regression testing** | Run timedemo + network test after each file | Catches issues immediately |
| **Truncation validation** | Add assertions checking `snprintf` return values | Detects truncation |
| **Parallel code paths** | `#ifdef SAFE_STRINGS` during transition | Easy comparison A/B |
| **Automated refactoring** | Use `sed`/`coccinelle` for mechanical replacements | Reduces human error |

**Implementation Detail**:
```c
/* Truncation detection wrapper: */
#define SAFE_SNPRINTF(buf, fmt, ...) do { \
    int _ret = snprintf(buf, sizeof(buf), fmt, ##__VA_ARGS__); \
    if (_ret >= (int)sizeof(buf)) { \
        Con_DPrintf("WARNING: truncation in %s:%d\n", __FILE__, __LINE__); \
    } \
} while(0)
```

**Applied to**: All files with `sprintf` starting with highest-risk files (`common.c`, `net_dgrm.c`, `sv_main.c`)

### M-T2: Assembly Removal — Gradual C Fallbacks

**Risk**: Performance regression when removing x86 assembly

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Conditional compilation** | `#ifdef USE_ASM` / `#else` C fallback | Zero risk — both paths compile |
| **Performance benchmarking** | Automated timedemo comparison in CI | Immediate regression detection |
| **SIMD intrinsics** | SSE2/NEON replacements for hottest loops | Approaches assembly performance |
| **Phased removal** | Remove least-critical assembly first | Limits performance impact |
| **Keep assembly option** | CMake `USE_ASM=ON` for x86 builds | No regression for x86 targets |

**Priority order for C fallback implementation**:
1. `math.s` (418 lines) — Standard math, easy to replace
2. `snd_mixa.s` (218 lines) — Simple mixing loop
3. `r_aliasa.s` (237 lines) — Model rendering
4. `d_draw.s` (1,037 lines) — Pixel drawing (with SSE2)
5. `d_polysa.s` (1,744 lines) — Polygon rasterization (last, most complex)

### M-T3: Headless Mode — Early Prototyping

**Risk**: Headless mode breaks server simulation

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Null driver testing** | Build and test with existing `vid_null.c`, `snd_null.c`, `in_null.c` first | Validates viability early |
| **Server-only mode** | Use QW dedicated server (`QW/server/`) as starting point | Already headless-capable |
| **Dependency mapping** | Trace all calls from `Host_Frame()` (`host.c:729`) to video/audio/input | Identifies hidden coupling |
| **Stub functions** | Implement minimal stubs that satisfy all callers | Prevents crashes |
| **Integration tests** | Connect real QW client to headless server, verify gameplay | Validates correctness |

**Dependency trace from Host_Frame**:
```
host.c:729 Host_Frame()
  ├── Host_ServerFrame() → sv_main.c (no video dependency ✓)
  ├── Host_ClientFrame() → cl_main.c
  │     ├── CL_ReadPackets() (no video dependency ✓)
  │     ├── CL_SendCmd() (no video dependency ✓)
  │     └── SCR_UpdateScreen() → screen.c → vid_*.c (DEPENDENCY ✗)
  ├── S_Update() → snd_dma.c (audio dependency ✗)
  └── IN_Commands() → in_*.c (input dependency ✗)
```

### M-T5: Memory Model — Container-Aware Configuration

**Risk**: Container memory limits conflict with engine pre-allocation

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Environment variable** | `QUAKE_MEMORY` controls allocation size | Aligns with container limits |
| **Dynamic sizing** | Read container memory limit from cgroup | Auto-configures |
| **Graceful OOM** | Check allocation result at `host.c:835` | Proper error message |
| **Memory budget** | Allocate max 80% of container limit | Leaves headroom for OS/runtime |

```c
/* Container-aware memory sizing: */
size_t get_available_memory(void) {
    /* Read from cgroup v2 */
    FILE *f = fopen("/sys/fs/cgroup/memory.max", "r");
    if (f) {
        size_t limit;
        if (fscanf(f, "%zu", &limit) == 1) {
            fclose(f);
            return limit * 80 / 100; /* 80% budget */
        }
        fclose(f);
    }
    return 16 * 1024 * 1024; /* Default 16MB */
}
```

### M-T7: Encoding Latency — Adaptive Quality

**Risk**: Video encoding exceeds latency budget

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Hardware encoding** | Use VAAPI/NVENC when available | 10x faster than software |
| **Adaptive resolution** | Reduce resolution if latency exceeds budget | Maintains responsiveness |
| **Adaptive framerate** | Drop to 30 FPS if 60 FPS exceeds budget | Halves encoding work |
| **Pre-allocated buffers** | Double-buffer framebuffer + encode buffer | Eliminates allocation stalls |
| **Encode-on-demand** | Skip encoding unchanged frames | Reduces average workload |

---

## 3. Schedule Risk Mitigations

### M-S1: Modularization Scope Creep — Time-Boxing

**Risk**: Phase 2 expands beyond 10 weeks

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Time-box each milestone** | Hard deadline per milestone, cut scope if needed | Prevents unbounded expansion |
| **Minimum viable interface** | Start with minimal interface, expand later | Limits initial scope |
| **Spike before commit** | 1-week prototype before committing to approach | Validates feasibility |
| **Scope reduction plan** | Pre-defined features to cut if behind schedule | Ready fallback |

**Scope reduction levels**:
1. **Full scope**: All 5 interfaces (platform, video, audio, input, renderer)
2. **Reduced**: 3 interfaces (platform, video, audio) — input uses existing null driver
3. **Minimal**: 1 interface (video only) — enough for headless mode

### M-S2: Security Fixes — Automation

**Risk**: 664 replacements take longer than estimated

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Coccinelle semantic patches** | Automated `sprintf→snprintf` transformation | 80% of replacements |
| **Batch processing** | Process one function type at a time across entire codebase | Consistent approach |
| **Pair review** | Two reviewers per security commit | Catches errors |
| **Acceptance criteria per file** | File-level checklist before merge | Ensures completeness |

**Coccinelle example**:
```
// Convert sprintf to snprintf
@@
expression buf, fmt;
expression list args;
@@
- sprintf(buf, fmt, args)
+ snprintf(buf, sizeof(buf), fmt, args)
```

---

## 4. Dependency Risk Mitigations

### M-D1: External Libraries — License Audit

**Risk**: Library compatibility with GPL v2

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Pre-approval list** | Maintain approved library list with licenses | Prevents issues |
| **License scanner** | Run Scancode in CI | Automated detection |
| **Vendoring** | Vendor critical dependencies | Stability |
| **Fallback alternatives** | Identify GPL-compatible alternatives for each dependency | Ready substitutes |

**License compatibility matrix**:
| Library | License | GPL v2 Compatible |
|---------|---------|-------------------|
| SDL2 | zlib | ✅ Yes |
| FFmpeg | LGPL 2.1 | ✅ Yes (dynamic linking) |
| libx264 | GPL v2 | ✅ Yes |
| cJSON | MIT | ✅ Yes |
| Unity (test) | MIT | ✅ Yes |
| libcurl | MIT/curl | ✅ Yes |

### M-D2: Game Data — Shareware Fallback

**Risk**: Proprietary game data unavailable for testing

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Shareware PAK** | Use freely distributable shareware `pak0.pak` | Full first episode |
| **Synthetic test data** | Create minimal test maps and models | Independence from game data |
| **Mock data** | Unit tests use in-memory mock data | No file dependency |

---

## 5. Operational Risk Mitigations

### M-O1: Cost Control — Resource Limits

**Risk**: Cloud compute costs escalate

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Budget alerts** | Azure cost alerts at 50%, 80%, 100% of budget | Early warning |
| **Auto-scale limits** | Max replicas cap in HPA | Prevents runaway scaling |
| **Spot instances** | Use Azure Spot VMs for non-critical sessions | 60-80% cost reduction |
| **Session timeout** | Auto-terminate idle sessions after 5 minutes | Reduces waste |
| **Reserved instances** | 1-year reservation for baseline capacity | 40% savings |

### M-O2: Security Incident — Defense in Depth

**Risk**: Undiscovered vulnerability exploited

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Container isolation** | Minimal capabilities, non-root, read-only FS | Limits blast radius |
| **Network policies** | Kubernetes NetworkPolicy restricting egress | Prevents data exfiltration |
| **Runtime security** | Falco/Sysdig for runtime monitoring | Detects exploitation |
| **Regular scanning** | Weekly Trivy scans of running images | Catches new CVEs |
| **Incident response plan** | Documented response procedures | Fast recovery |

### M-O3: Knowledge Gap — Documentation & Training

**Risk**: Team lacks expertise across all required technologies

| Strategy | Implementation | Effectiveness |
|----------|---------------|---------------|
| **Architecture Decision Records** | Document every technical decision | Preserves context |
| **Code walkthroughs** | Weekly sessions on codebase areas | Knowledge sharing |
| **Existing reverse engineering docs** | `specs/docs/`, `specs/features/` | Comprehensive reference |
| **Pair programming** | Senior + junior on complex tasks | Skill transfer |
| **External consultants** | Azure/K8s specialists for Phase 4 | Targeted expertise |

---

## 6. Feature Flag Strategy

Use feature flags to control rollout of new functionality:

| Flag | Default | Controls |
|------|---------|----------|
| `USE_ASM` | ON (x86) / OFF (other) | Assembly vs. C rendering |
| `USE_INTERFACES` | OFF → ON after validation | New module interfaces |
| `HEADLESS_MODE` | OFF | Headless video/audio/input |
| `STRUCTURED_LOGGING` | OFF → ON in containers | JSON vs. text logging |
| `HEALTH_ENDPOINTS` | OFF → ON in containers | HTTP health probe server |
| `ENV_CONFIG` | OFF → ON in containers | Environment variable configuration |

**Implementation**: CMake options + compile-time `#ifdef`

```cmake
option(USE_ASM "Use x86 assembly optimizations" ON)
option(HEADLESS_MODE "Build with headless mode support" OFF)
option(STRUCTURED_LOGGING "Enable JSON structured logging" OFF)
option(HEALTH_ENDPOINTS "Enable HTTP health endpoints" OFF)
```

---

## 7. Canary Deployment Strategy

For Phase 4 (Azure deployment):

```
1. Deploy new version to 1 pod (canary)
2. Route 5% of traffic to canary
3. Monitor for 1 hour:
   - Error rate < 1%
   - Latency P99 < 50ms
   - No crash restarts
4. If healthy: increase to 25% traffic
5. Monitor for 4 hours
6. If healthy: increase to 100% (full rollout)
7. If unhealthy at any stage: rollback canary
```

**Kubernetes implementation**:
```yaml
# Canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quake-engine-canary
spec:
  replicas: 1  # Single canary pod
  selector:
    matchLabels:
      app: quake-engine
      track: canary
```
