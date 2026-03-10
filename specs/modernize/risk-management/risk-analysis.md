# Risk Analysis — Quake Engine Modernization

> Risk identification: technical risks, schedule risks, dependency risks.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document identifies and assesses the key risks associated with modernizing the 1996 Quake engine into a cloud-native application. Risks are categorized as **Technical**, **Schedule**, **Dependency**, and **Operational**, with impact and likelihood ratings for each.

---

## 2. Risk Rating Scale

| Rating | Likelihood | Impact |
|--------|-----------|--------|
| **Critical** | > 80% | Project failure |
| **High** | 50-80% | Major delay or scope reduction |
| **Medium** | 20-50% | Moderate delay or workaround needed |
| **Low** | < 20% | Minor inconvenience |

---

## 3. Technical Risks

### T1: Security Remediation Introduces Bugs

| Attribute | Value |
|-----------|-------|
| **Likelihood** | High (60%) |
| **Impact** | High |
| **Risk Score** | **Critical** |
| **Description** | Replacing 664 string operations risks introducing truncation bugs, off-by-one errors, or behavioral changes. For example, `sprintf` at `Quake/WinQuake/common.c:1286` constructs file paths — truncation could cause file-not-found errors. |
| **Affected Files** | All 240 `.c` files with string operations |
| **Indicators** | Test failures, file loading errors, network protocol mismatches |

### T2: Assembly Removal Causes Performance Regression

| Attribute | Value |
|-----------|-------|
| **Likelihood** | High (70%) |
| **Impact** | Medium |
| **Risk Score** | **High** |
| **Description** | The 21 assembly files (10,748 lines) in `Quake/WinQuake/*.s` provide 2-5x speedup for rendering inner loops. C replacements will be slower. Key files: `d_polysa.s` (1,744 lines), `d_draw.s` (1,037 lines). |
| **Affected Files** | All `.s` files and their C counterparts |
| **Indicators** | FPS drop > 50% in timedemo benchmark |

### T3: Headless Mode Breaks Server Simulation

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (40%) |
| **Impact** | High |
| **Risk Score** | **High** |
| **Description** | The engine's server logic (`sv_main.c`, `sv_phys.c`) is tightly coupled to the frame loop in `host.c:729` (Host_Frame). Running without video/audio initialization may expose hidden dependencies on display timing or buffer allocation in `vid_win.c:260` (VID_AllocBuffers). |
| **Affected Files** | `host.c`, `sv_main.c`, `vid_*.c`, `snd_*.c` |
| **Indicators** | Server crashes, physics glitches, timing errors |

### T4: Platform Abstraction Overhead

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Low (15%) |
| **Impact** | Medium |
| **Risk Score** | **Low** |
| **Description** | Adding function pointer indirection for platform calls adds overhead to hot paths. If the renderer interface `renderer_interface_t` adds per-pixel indirection (e.g., in `d_scan.c:274` D_DrawSpans8), performance could degrade significantly. |
| **Affected Files** | All interface implementations |
| **Indicators** | Measurable FPS drop in profiling |

### T5: Memory Model Incompatibility

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (35%) |
| **Impact** | High |
| **Risk Score** | **High** |
| **Description** | The Zone/Hunk/Cache memory system (`zone.c`) manages a single contiguous memory block. Container memory limits may conflict with this model — the engine pre-allocates all memory at startup. If container memory limit is below the engine's request, startup fails. |
| **Affected Files** | `zone.c`, `host.c:835` (Host_Init memory allocation) |
| **Indicators** | Container OOMKill, startup failures |

### T6: Network Protocol Breaking Changes

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (30%) |
| **Impact** | High |
| **Risk Score** | **Medium** |
| **Description** | Modifying network code (e.g., replacing `sprintf` in `net_dgrm.c:134` for address formatting) could change packet format, breaking compatibility with existing Quake clients. |
| **Affected Files** | `net_dgrm.c`, `QW/server/sv_ents.c:155`, `QW/client/cl_ents.c:265` |
| **Indicators** | Client connection failures, entity desync |

### T7: Video Encoding Latency Budget

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (45%) |
| **Impact** | High |
| **Risk Score** | **High** |
| **Description** | The 50ms end-to-end latency target for game streaming requires frame extraction (~1ms) + encoding (~5-16ms) + network (~5ms) + decode (~5ms) + display (~5ms). If any component exceeds budget, the experience becomes unplayable. Software H.264 encoding at 640×480 may take 10-20ms per frame. |
| **Affected Files** | New streaming pipeline code |
| **Indicators** | Perceived input lag, motion sickness |

---

## 4. Schedule Risks

### S1: Scope Creep in Modularization

| Attribute | Value |
|-----------|-------|
| **Likelihood** | High (65%) |
| **Impact** | High |
| **Risk Score** | **Critical** |
| **Description** | Modularizing the engine (Phase 2, estimated 10 weeks) could expand as hidden dependencies between subsystems are discovered. The 105+ global variables create implicit coupling that isn't visible until refactoring begins. |
| **Indicators** | Task estimates consistently exceeded, cascading changes |

### S2: Security Fixes Take Longer Than Estimated

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (40%) |
| **Impact** | Medium |
| **Risk Score** | **Medium** |
| **Description** | While 664 string replacements seem mechanical, each requires understanding the buffer context, validating truncation behavior, and ensuring the replacement doesn't break string protocol expectations. |
| **Indicators** | Rate of fixes below 50/day, frequent rework |

### S3: Azure Deployment Complexity

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (35%) |
| **Impact** | Medium |
| **Risk Score** | **Medium** |
| **Description** | AKS configuration, networking (UDP game traffic + WebSocket streaming), and auto-scaling require specialized knowledge. UDP in Kubernetes has known challenges with service meshes and load balancers. |
| **Indicators** | Networking issues, scaling failures |

---

## 5. Dependency Risks

### D1: External Library Compatibility

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Low (20%) |
| **Impact** | Medium |
| **Risk Score** | **Low** |
| **Description** | New dependencies (SDL2, FFmpeg, libx264) may have version conflicts or licensing issues with GPL v2. |
| **Affected** | All new external dependencies |
| **Indicators** | Build failures, license audit findings |

### D2: Game Data Availability

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (30%) |
| **Impact** | High |
| **Risk Score** | **Medium** |
| **Description** | The engine requires `pak0.pak` game data files which are proprietary. Without game data, testing is limited to engine startup verification. The shareware data is freely distributable. |
| **Indicators** | Cannot run full integration or E2E tests |

### D3: Kubernetes Version Compatibility

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Low (15%) |
| **Impact** | Low |
| **Risk Score** | **Low** |
| **Description** | AKS Kubernetes version updates may break manifests or RBAC configurations. |
| **Indicators** | Deployment failures after AKS upgrade |

---

## 6. Operational Risks

### O1: Cost Overrun in Cloud

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (40%) |
| **Impact** | Medium |
| **Risk Score** | **Medium** |
| **Description** | Each game session requires CPU for rendering + encoding. At scale, compute costs may be prohibitive. 100 concurrent sessions at Standard_D4s_v3 pricing could cost $2,000+/month. |
| **Indicators** | Azure billing alerts, cost-per-session exceeding $0.50/hour |

### O2: Security Incident in Production

| Attribute | Value |
|-----------|-------|
| **Likelihood** | Medium (30%) |
| **Impact** | Critical |
| **Risk Score** | **High** |
| **Description** | Even after remediation, the C codebase may contain undiscovered vulnerabilities. A buffer overflow exploit in the network-facing code could compromise the container or cluster. |
| **Indicators** | Anomalous traffic, crash loops, unexpected processes |

### O3: Talent/Knowledge Gap

| Attribute | Value |
|-----------|-------|
| **Likelihood** | High (55%) |
| **Impact** | Medium |
| **Risk Score** | **High** |
| **Description** | The project requires expertise in 1996 C code, x86 assembly, Kubernetes, Azure, video encoding, and WebSocket streaming. Finding team members with this combination is challenging. |
| **Indicators** | Slow progress, frequent questions, incorrect implementations |

---

## 7. Risk Summary Matrix

```
Impact ▲
       │
Critical ├─ O2 ──────────────────── T1, S1
       │
  High ├─ T5, T6 ──── T2, T3, T7 ── O3
       │
Medium ├─ D3 ─── D1 ── S2, S3, O1 ── T4
       │
  Low  ├───────────────────────────────
       │
       └──Low───Medium────High───Critical──▶ Likelihood
```

---

## 8. Top 5 Risks (Priority Order)

| Rank | Risk | Score | Primary Mitigation |
|------|------|-------|-------------------|
| 1 | **T1**: Security fixes introduce bugs | Critical | Comprehensive test suite, incremental commits |
| 2 | **S1**: Modularization scope creep | Critical | Strict phase gates, time-boxing |
| 3 | **T3**: Headless mode breaks simulation | High | Early prototyping, headless-first testing |
| 4 | **T7**: Encoding latency budget | High | Hardware encoding, adaptive quality |
| 5 | **O3**: Talent/knowledge gap | High | Documentation, pair programming, phased training |
