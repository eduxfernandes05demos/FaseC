# Modernization Roadmap — Quake Engine to Cloud-Native

> Overall modernization roadmap with phases, milestones, and dependencies.
> All references point to actual source files in this repository.

---

## 1. Vision

Transform the 1996 Quake engine from a monolithic, platform-specific C application into a cloud-native, containerized game streaming service deployable on Azure Kubernetes Service (AKS) or Azure Container Apps (ACA).

```
Phase 0          Phase 1          Phase 2          Phase 3          Phase 4
Foundation   →   Hardening   →   Modularization  →  Cloud-Ready  →  Cloud-Native
(4 weeks)        (4 weeks)        (10 weeks)         (6 weeks)       (8 weeks)
```

**Total Estimated Duration**: 32 weeks (8 months)

---

## 2. Phase 0 — Foundation (Weeks 1-4)

**Objective**: Establish modern build infrastructure and development tooling.

### Milestones

| ID | Milestone | Deliverable | Effort |
|----|-----------|-------------|--------|
| M0.1 | CMake build system | `CMakeLists.txt` replacing all Makefiles and `.dsp` files | 2 weeks |
| M0.2 | CI/CD pipeline | GitHub Actions workflow for build + test | 1 week |
| M0.3 | Test framework | CUnit/Unity integration with first tests | 1 week |

### Dependencies

```
M0.1 (CMake) → M0.2 (CI/CD) → M0.3 (Tests)
```

### Key Files Affected

| Current File | Action |
|-------------|--------|
| `Quake/WinQuake/Makefile.linuxi386` | Replace with CMake |
| `Quake/WinQuake/Makefile.Solaris` | Replace with CMake |
| `Quake/QW/Makefile.Linux` | Replace with CMake |
| `Quake/WinQuake/WinQuake.dsp` | Replace with CMake |
| `Quake/QW/qw.dsw` | Replace with CMake |

### Exit Criteria

- [ ] `cmake --build .` compiles WinQuake and QuakeWorld on Linux
- [ ] GitHub Actions runs build on every push
- [ ] At least 10 unit tests pass in CI

---

## 3. Phase 1 — Security Hardening (Weeks 5-8)

**Objective**: Eliminate critical security vulnerabilities blocking network deployment.

### Milestones

| ID | Milestone | Deliverable | Effort |
|----|-----------|-------------|--------|
| M1.1 | Buffer overflow remediation | Replace 664 unsafe string operations | 3 weeks |
| M1.2 | Memory safety fixes | Add NULL checks, remove system() calls | 0.5 weeks |
| M1.3 | Security test suite | Fuzzing harness + buffer overflow tests | 0.5 weeks |

### Dependencies

```
M0.1 (CMake) → M1.1 (Buffer fixes) → M1.3 (Security tests)
M0.3 (Tests) → M1.3 (Security tests)
```

### Key Files Affected

- **362 files** with sprintf → snprintf
- **232 files** with strcpy → strncpy/strlcpy
- **70 files** with strcat → strncat/strlcat
- `Quake/WinQuake/sys_linux.c:276` — Remove system() call
- `Quake/QW/client/sys_linux.c:278` — Remove system() call

### Exit Criteria

- [ ] Zero instances of sprintf/strcpy/strcat in codebase
- [ ] All malloc calls have NULL checks
- [ ] No system() calls remain
- [ ] Security fuzzing tests pass with no crashes

---

## 4. Phase 2 — Modularization (Weeks 9-18)

**Objective**: Decouple platform, rendering, audio, and input into pluggable modules.

### Milestones

| ID | Milestone | Deliverable | Effort |
|----|-----------|-------------|--------|
| M2.1 | Platform abstraction layer | `platform_interface_t` with runtime selection | 3 weeks |
| M2.2 | Headless video driver | `vid_headless.c` for rendering without display | 2 weeks |
| M2.3 | Headless audio/input | `snd_null.c` + `in_null.c` enhanced for headless | 1 week |
| M2.4 | Renderer modularization | `renderer_interface_t` with software/GL/null backends | 3 weeks |
| M2.5 | Global state encapsulation | Engine state structs replacing globals | 1 week |

### Dependencies

```
M1.1 (Security) → M2.1 (Platform) → M2.2 (Headless Video)
                                    → M2.3 (Headless Audio)
M2.1 (Platform) → M2.4 (Renderer)
M2.4 (Renderer) → M2.5 (State)
```

### Key Files Affected

| Module | Current Files | New Abstraction |
|--------|--------------|-----------------|
| Platform | `sys_win.c`, `sys_linux.c`, `sys_dos.c` | `platform_interface_t` |
| Video | `vid_win.c`, `vid_svgalib.c`, `vid_x.c` | `video_interface_t` |
| Audio | `snd_win.c`, `snd_linux.c`, `snd_dos.c` | `audio_interface_t` |
| Input | `in_win.c`, `in_dos.c`, `in_sun.c` | `input_interface_t` |
| Renderer | `r_main.c`, `gl_rmain.c` | `renderer_interface_t` |

### Exit Criteria

- [ ] Engine runs in headless mode (no display, no audio, no input hardware)
- [ ] Platform backend selected at runtime via configuration
- [ ] Renderer backend selected at runtime
- [ ] Engine state contained in structs (no globals in new code)

---

## 5. Phase 3 — Cloud-Ready (Weeks 19-24)

**Objective**: Containerize the engine and add cloud operational requirements.

### Milestones

| ID | Milestone | Deliverable | Effort |
|----|-----------|-------------|--------|
| M3.1 | Containerization | Dockerfile + docker-compose | 2 weeks |
| M3.2 | Health & observability | Health endpoints, structured logging, metrics | 2 weeks |
| M3.3 | Configuration management | Environment variable support, config maps | 1 week |
| M3.4 | Graceful shutdown | SIGTERM handling, connection draining | 1 week |

### Dependencies

```
M2.2 (Headless) → M3.1 (Container) → M3.2 (Health)
M2.3 (Audio/Input Headless) → M3.1 (Container)
M3.1 (Container) → M3.3 (Config)
M3.1 (Container) → M3.4 (Shutdown)
```

### Key Deliverables

| Deliverable | Description |
|-------------|-------------|
| `Dockerfile` | Multi-stage build: build + runtime images |
| `docker-compose.yml` | Local development orchestration |
| Health HTTP endpoint | `/healthz`, `/readyz` for Kubernetes probes |
| Structured logging | JSON to stdout with log levels and correlation IDs |
| SIGTERM handler | Graceful shutdown with connection draining |

### Exit Criteria

- [ ] `docker build` produces working container image
- [ ] Container starts in headless mode and accepts connections
- [ ] Health endpoints respond to HTTP probes
- [ ] SIGTERM causes graceful shutdown within 30 seconds
- [ ] All configuration via environment variables

---

## 6. Phase 4 — Cloud-Native Streaming (Weeks 25-32)

**Objective**: Deploy on Azure with game streaming capability.

### Milestones

| ID | Milestone | Deliverable | Effort |
|----|-----------|-------------|--------|
| M4.1 | Session manager | Multi-instance orchestration service | 2 weeks |
| M4.2 | Streaming gateway | WebSocket/WebRTC API for frame delivery | 3 weeks |
| M4.3 | Frame encoding pipeline | Pixel buffer → H.264 encoding | 2 weeks |
| M4.4 | Azure deployment | AKS/ACA + IaC (Bicep) + monitoring | 1 week |

### Dependencies

```
M3.1 (Container) → M4.1 (Session Manager) → M4.4 (Azure)
M3.1 (Container) → M4.2 (Gateway) → M4.4 (Azure)
M2.2 (Headless) → M4.3 (Encoding) → M4.2 (Gateway)
```

### Exit Criteria

- [ ] Multiple game sessions run concurrently in containers
- [ ] Clients connect via WebSocket and receive streamed frames
- [ ] Frame latency < 50ms end-to-end
- [ ] Azure deployment via `bicep deploy` or `terraform apply`
- [ ] Auto-scaling based on session demand

---

## 7. Dependency Graph

```
M0.1 CMake ──────────────────────────────────────────────────┐
  │                                                           │
  ├─→ M0.2 CI/CD                                             │
  │     │                                                     │
  │     └─→ M0.3 Tests                                       │
  │           │                                               │
  ├─→ M1.1 Buffer Fixes ─→ M1.3 Security Tests               │
  │     │                                                     │
  │     └─→ M2.1 Platform Abstraction                         │
  │           │                                               │
  │           ├─→ M2.2 Headless Video ─→ M3.1 Container       │
  │           │                            │                  │
  │           ├─→ M2.3 Headless Audio ─────┘                  │
  │           │                            │                  │
  │           └─→ M2.4 Renderer            ├─→ M3.2 Health    │
  │                 │                      ├─→ M3.3 Config    │
  │                 └─→ M2.5 State         ├─→ M3.4 Shutdown  │
  │                                        │                  │
  │                                        ├─→ M4.1 Sessions  │
  │                                        ├─→ M4.2 Gateway   │
  │                                        └─→ M4.3 Encoding  │
  │                                              │            │
  │                                              └─→ M4.4 Azure
  └───────────────────────────────────────────────────────────┘
```

---

## 8. Risk-Adjusted Timeline

| Phase | Optimistic | Expected | Pessimistic |
|-------|-----------|----------|-------------|
| Phase 0 — Foundation | 3 weeks | 4 weeks | 6 weeks |
| Phase 1 — Hardening | 3 weeks | 4 weeks | 6 weeks |
| Phase 2 — Modularization | 8 weeks | 10 weeks | 14 weeks |
| Phase 3 — Cloud-Ready | 4 weeks | 6 weeks | 8 weeks |
| Phase 4 — Cloud-Native | 6 weeks | 8 weeks | 12 weeks |
| **Total** | **24 weeks** | **32 weeks** | **46 weeks** |
