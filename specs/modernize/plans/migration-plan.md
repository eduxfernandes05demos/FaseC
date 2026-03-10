# Migration Plan — Quake Engine to Cloud-Native

> Step-by-step migration approach from monolithic C to containerized microservices on Azure.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document provides the step-by-step migration plan for transforming the Quake engine from a monolithic C application into a set of containerized microservices deployed on Azure. The plan follows a **strangler fig pattern** — wrapping the existing engine rather than rewriting it — to minimize risk and maintain a working application at every stage.

---

## 2. Migration Principles

1. **Preserve functionality**: The engine must remain playable throughout migration
2. **Incremental delivery**: Each step produces a shippable artifact
3. **Reversible changes**: Every step can be rolled back independently
4. **Test-driven**: Each change is validated by automated tests before proceeding
5. **No big bang**: Never change more than one subsystem at a time

---

## 3. Step-by-Step Migration

### Step 1: Build System Migration (Week 1-2)

**Objective**: Replace fragmented Makefiles/MSVC projects with unified CMake.

| Action | Files Affected | Validation |
|--------|---------------|-----------|
| Create root `CMakeLists.txt` | New file at `Quake/CMakeLists.txt` | `cmake --build .` succeeds |
| Create WinQuake CMake target | Replaces `Quake/WinQuake/Makefile.linuxi386` | Binary runs identical to Makefile build |
| Create QW client CMake target | Replaces `Quake/QW/Makefile.Linux` (client part) | Binary runs identical |
| Create QW server CMake target | Replaces `Quake/QW/Makefile.Linux` (server part) | Binary runs identical |
| Add assembly toggle | `option(USE_ASM "Use x86 assembly" ON)` | Builds with and without assembly |
| Keep original build files | Move to `Quake/legacy-build/` | Original builds still work |

**Exit Criteria**:
- [ ] CMake builds all three targets (WinQuake, QW client, QW server) on Linux
- [ ] Binary output matches original Makefile builds
- [ ] CI pipeline runs CMake build on push

### Step 2: CI/CD Foundation (Week 3)

**Objective**: Establish automated build and test pipeline.

| Action | Validation |
|--------|-----------|
| Create `.github/workflows/ci.yml` | Workflow passes on push |
| Add CTest integration | `ctest` runs successfully |
| Add initial unit tests (10+) | Tests for `mathlib.c`, `common.c`, `zone.c`, `cvar.c` |
| Add compiler warnings as errors | `-Wall -Wextra -Werror` with known exceptions |

**Exit Criteria**:
- [ ] Green CI on every push to main
- [ ] At least 10 unit tests running
- [ ] Build fails on new warnings

### Step 3: Security Remediation (Week 4-7)

**Objective**: Eliminate all critical security vulnerabilities.

| Sub-step | Scope | Approach |
|----------|-------|---------|
| 3a: Implement `Q_strlcpy()` / `Q_strlcat()` | New utility functions | Add to `common.c` or new `safe_str.c` |
| 3b: Replace `sprintf` in WinQuake | 180+ instances | Mechanical `sprintf` → `snprintf` |
| 3c: Replace `sprintf` in QW client | 100+ instances | Same mechanical replacement |
| 3d: Replace `sprintf` in QW server | 80+ instances | Same mechanical replacement |
| 3e: Replace `strcpy`/`strcat` | 302 instances | Replace with `Q_strlcpy`/`Q_strlcat` |
| 3f: Add malloc NULL checks | 9+ instances | Add error handling after each `malloc` |
| 3g: Remove `system()` calls | 2 instances | Remove at `sys_linux.c:276` and `QW/client/sys_linux.c:278` |
| 3h: Add security tests | New test file | Fuzzing harness + buffer tests |

**Exit Criteria**:
- [ ] Zero instances of `sprintf`, `strcpy`, `strcat` (unbounded) in codebase
- [ ] `grep -r "sprintf\|strcpy\|strcat" Quake/ --include="*.c"` returns zero unbounded calls
- [ ] All malloc calls have NULL checks
- [ ] Security fuzzing runs without crashes

### Step 4: Platform Abstraction (Week 8-10)

**Objective**: Decouple platform-specific code into pluggable interfaces.

| Sub-step | Action | Files |
|----------|--------|-------|
| 4a: Define interfaces | Create header files for each interface | New `platform_interface.h`, `video_interface.h`, etc. |
| 4b: Wrap existing implementations | Adapt `sys_linux.c`, `vid_svgalib.c`, etc. to match interfaces | Existing platform files |
| 4c: Create headless backends | `sys_headless.c`, `vid_headless.c` (new) | New files |
| 4d: Runtime backend selection | Add startup option `--backend=headless` | `host.c:835` (Host_Init) |

**Key Files**:
- `Quake/WinQuake/sys_linux.c` → Wrap as `platform_linux` implementation
- `Quake/WinQuake/vid_svgalib.c` (line 555: `VID_Init()`) → Wrap behind `video_interface_t`
- `Quake/WinQuake/vid_null.c` → Enhance for headless mode
- `Quake/WinQuake/snd_null.c` → Enhance with audio capture
- `Quake/WinQuake/in_null.c` → Enhance with remote input injection

**Exit Criteria**:
- [ ] Engine starts with `--backend=headless` (no display, no audio, no input hardware)
- [ ] Engine starts with `--backend=linux` (original behavior preserved)
- [ ] Headless mode runs server simulation correctly

### Step 5: Containerization (Week 11-13)

**Objective**: Package the engine as a container image.

| Sub-step | Action | Deliverable |
|----------|--------|-------------|
| 5a: Create Dockerfile | Multi-stage build | `Dockerfile` |
| 5b: Create docker-compose | Local development setup | `docker-compose.yml` |
| 5c: Add health endpoints | HTTP `/healthz`, `/readyz` | New `health.c` |
| 5d: Add structured logging | JSON output to stdout | Modified `console.c` |
| 5e: Add graceful shutdown | SIGTERM handler | Modified `host.c` |
| 5f: Environment variable config | Replace hard-coded values | Modified `host.c`, `net_main.c` |

**Configuration Migration**:
| Hard-coded | Environment Variable | Source |
|-----------|---------------------|--------|
| Port 26000 | `QUAKE_PORT` | `net_main.c:34` |
| `"id1"` game dir | `QUAKE_GAMEDIR` | `common.c:1775` |
| Memory size | `QUAKE_MEMORY` | Host_Init parameter |

**Exit Criteria**:
- [ ] `docker build` succeeds
- [ ] `docker run` starts headless engine
- [ ] Health endpoints respond
- [ ] SIGTERM causes graceful shutdown
- [ ] All configuration via environment variables

### Step 6: Frame Extraction & Encoding (Week 14-16)

**Objective**: Enable frame buffer extraction and video encoding.

| Sub-step | Action |
|----------|--------|
| 6a: Framebuffer API | `VID_GetFramebuffer()` in headless video driver |
| 6b: Color space conversion | RGB → YUV420 conversion |
| 6c: Video encoding integration | FFmpeg/libx264 encoding pipeline |
| 6d: Audio capture | PCM capture from sound mixer |

**Key Integration Points**:
- Software renderer writes to `vid.buffer` → Extract after `R_RenderView()` (`r_main.c:1049`)
- Sound mixer output at `S_PaintChannels()` (`snd_mix.c:261`) → Capture mixed audio

**Exit Criteria**:
- [ ] Can extract rendered frames at 30+ FPS
- [ ] Frames encode to H.264 with < 16ms latency
- [ ] Audio captures to PCM stream

### Step 7: Session Manager (Week 17-18)

**Objective**: Manage multiple concurrent game sessions.

| Action | Deliverable |
|--------|-------------|
| REST API for session lifecycle | `POST /sessions`, `GET /sessions`, `DELETE /sessions/{id}` |
| Container orchestration integration | Kubernetes API for pod management |
| Session state tracking | In-memory state + optional Redis persistence |
| Health monitoring | Track session health via engine health endpoints |

**Exit Criteria**:
- [ ] Can create, list, and destroy sessions via REST API
- [ ] Each session runs in its own container
- [ ] Failed sessions are detected and cleaned up

### Step 8: Streaming Gateway (Week 19-22)

**Objective**: Build WebSocket/WebRTC gateway for game streaming.

| Action | Deliverable |
|--------|-------------|
| WebSocket server | Accept client connections |
| Frame relay | Receive encoded frames from engine, stream to clients |
| Input relay | Receive client input, inject into engine |
| Audio relay | Stream captured audio to clients |

**Exit Criteria**:
- [ ] Client connects via WebSocket
- [ ] Video stream renders in browser
- [ ] Input from browser controls game
- [ ] Audio streams to client
- [ ] End-to-end latency < 50ms

### Step 9: Azure Deployment (Week 23-24)

**Objective**: Deploy complete stack on Azure.

| Action | Deliverable |
|--------|-------------|
| Bicep/Terraform IaC | Azure resource definitions |
| AKS cluster setup | Kubernetes cluster with auto-scaling |
| ACR integration | Container registry with CI push |
| Monitoring setup | Azure Monitor + Log Analytics |
| DNS + Ingress | Public endpoint with TLS |

**Exit Criteria**:
- [ ] `az deployment create` provisions all resources
- [ ] Application accessible via public URL
- [ ] Auto-scaling works based on session count
- [ ] Monitoring dashboards show key metrics

---

## 4. Data Migration

The Quake engine has minimal persistent data:

| Data | Current Storage | Target Storage | Migration |
|------|----------------|----------------|-----------|
| Game assets (PAK files) | Local filesystem | Azure Blob Storage / Container volume | Copy to persistent volume |
| Config files | Local filesystem | ConfigMaps + environment variables | Rewrite configuration loading |
| Save games | Local filesystem | Azure Blob Storage | Implement save/load API |
| Demo recordings | Local filesystem | Azure Blob Storage | Implement recording API |

---

## 5. Migration Validation Gates

Each step has a **go/no-go decision gate** before proceeding:

| Gate | Criteria | Decision |
|------|----------|----------|
| G1 (After Step 1) | CMake builds match original | Proceed if binaries match |
| G2 (After Step 3) | Zero security scan findings | Proceed if clean |
| G3 (After Step 4) | Headless mode works | Proceed if server simulates correctly |
| G4 (After Step 5) | Container runs stably for 24h | Proceed if no crashes |
| G5 (After Step 6) | Encoding latency < 16ms | Proceed if meets budget |
| G6 (After Step 8) | E2E streaming works | Proceed if playable |
| G7 (After Step 9) | Production deployment stable | Release |
