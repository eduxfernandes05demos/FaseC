# Validation Criteria — Quake Engine Modernization

> Success criteria: measurable KPIs for each modernization phase.
> All references point to actual source files in this repository.

---

## 1. Executive Summary

This document defines the measurable success criteria and Key Performance Indicators (KPIs) for each phase of the modernization. Every criterion has a specific metric, measurement method, and pass/fail threshold. No phase is considered complete until all its criteria are met.

---

## 2. Phase 0 — Foundation Validation

### Build System (CMake)

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Build success | All targets compile | 100% | `cmake --build . 2>&1 | grep -c "error"` = 0 |
| Binary equivalence | Output matches original build | Functional match | Both binaries complete timedemo identically |
| Build time | CMake build time vs Makefile | ≤ 120% of original | `time cmake --build .` vs `time make` |
| Cross-platform | Builds on Linux x86_64 | Pass/Fail | CI green on ubuntu-latest |
| Assembly toggle | Builds with `USE_ASM=OFF` | Pass/Fail | `cmake -DUSE_ASM=OFF && cmake --build .` succeeds |

### CI/CD Pipeline

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Pipeline reliability | CI passes on clean commit | 100% | 10 consecutive green builds |
| Pipeline speed | Time from push to result | < 10 minutes | GitHub Actions timing |
| Test execution | Unit tests run in CI | All pass | CTest results in CI log |

### Testing Framework

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Test count | Number of unit tests | ≥ 10 | `ctest --test-dir build -N | tail -1` |
| Test coverage | Lines covered by tests | ≥ 5% of core modules | gcov/lcov report |
| Test reliability | Tests pass consistently | 100% (no flaky tests) | 10 consecutive runs |

---

## 3. Phase 1 — Security Hardening Validation

### Buffer Overflow Remediation

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| sprintf elimination | Instances of unbounded sprintf | 0 | `grep -rn "sprintf\b" legacy-src/ --include="*.c" | grep -v snprintf | wc -l` |
| strcpy elimination | Instances of unbounded strcpy | 0 | `grep -rn "\bstrcpy\b" legacy-src/ --include="*.c" | grep -v strncpy | wc -l` |
| strcat elimination | Instances of unbounded strcat | 0 | `grep -rn "\bstrcat\b" legacy-src/ --include="*.c" | grep -v strncat | wc -l` |
| vsprintf elimination | Instances of vsprintf | 0 | `grep -rn "vsprintf\b" legacy-src/ --include="*.c" | grep -v vsnprintf | wc -l` |

### Memory Safety

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Malloc NULL checks | Unchecked malloc calls | 0 | Static analysis (Cppcheck) |
| system() removal | Instances of system() | 0 | `grep -rn "system(" legacy-src/ --include="*.c" | wc -l` |
| ASan clean | AddressSanitizer findings | 0 | Run timedemo with `-fsanitize=address` |
| Fuzzing stability | Crashes during fuzzing | 0 in 1 hour | AFL++ with network protocol corpus |

### Functional Preservation

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Timedemo completion | Demo completes without crash | Pass/Fail | `./quake -timedemo demo1` exits cleanly |
| FPS preservation | Timedemo FPS after security fixes | ≥ 95% of baseline | Compare FPS output |
| Network connectivity | Client-server connection | Pass/Fail | Connect QW client to QW server |

---

## 4. Phase 2 — Modularization Validation

### Platform Abstraction

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Interface completeness | All platform functions abstracted | 100% | Code review of interface headers |
| Backend count | Number of working backends | ≥ 2 (Linux + headless) | Build and test each backend |
| Runtime selection | Backend selected at startup | Pass/Fail | `./quake --backend=headless` works |

### Headless Mode

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Starts without display | Engine starts on headless server | Pass/Fail | Run on server without X11 |
| Server simulation | Server physics runs correctly | Pass/Fail | Connect client, verify movement |
| Frame generation | Framebuffer populated | Pass/Fail | Extract frame, verify non-black pixels |
| CPU usage | Headless CPU consumption | ≤ 1 core at idle | `top` measurement |
| Memory usage | Headless memory footprint | ≤ 64 MB | `pmap` or `/proc/pid/status` |

### Renderer Modularization

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Software renderer | Renders correctly via interface | Pass/Fail | Visual comparison with pre-modularization |
| OpenGL renderer | Renders correctly via interface | Pass/Fail | Visual comparison |
| Null renderer | Runs without rendering | Pass/Fail | CPU time in render = 0 |
| Performance overhead | Interface call overhead | ≤ 1% FPS loss | Timedemo comparison |

---

## 5. Phase 3 — Container Validation

### Containerization

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Image build | Dockerfile builds successfully | Pass/Fail | `docker build .` exits 0 |
| Image size | Container image size | ≤ 200 MB | `docker images | grep quake` |
| Startup time | Time from `docker run` to ready | ≤ 5 seconds | Measure time to health endpoint response |
| Stability | Continuous run without crash | ≥ 24 hours | Long-running stability test |
| Non-root | Runs as non-root user | Pass/Fail | `docker exec ... whoami` ≠ root |

### Health & Observability

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Liveness probe | `/healthz` responds | 200 OK | `curl http://localhost:8080/healthz` |
| Readiness probe | `/readyz` responds when ready | 200 OK after init | `curl http://localhost:8080/readyz` |
| Structured logging | JSON log output | Valid JSON | `docker logs ... | jq .` succeeds |
| Log levels | DEBUG/INFO/WARN/ERROR present | All levels used | Grep logs for each level |
| Metrics export | Prometheus metrics endpoint | Pass/Fail | `curl http://localhost:8080/metrics` |

### Configuration

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Port config | `QUAKE_PORT` env var works | Pass/Fail | Start with custom port, verify listening |
| Game dir config | `QUAKE_GAMEDIR` env var works | Pass/Fail | Start with custom game dir |
| Memory config | `QUAKE_MEMORY` env var works | Pass/Fail | Verify allocated memory matches config |

### Graceful Shutdown

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| SIGTERM handling | Process exits on SIGTERM | Exit code 0 | `docker stop <container>` → exit 0 |
| Shutdown time | Time from SIGTERM to exit | ≤ 30 seconds | Measure with `time docker stop` |
| Connection draining | Active connections complete | Pass/Fail | Connect client, send SIGTERM, verify clean disconnect |

---

## 6. Phase 4 — Cloud-Native Validation

### Session Management

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Session creation | REST API creates session | ≤ 10 seconds | `time curl -X POST /sessions` |
| Session listing | REST API lists sessions | Pass/Fail | `curl GET /sessions` returns JSON |
| Session cleanup | Expired sessions removed | Within 60 seconds | Wait after session ends, verify cleanup |
| Concurrent sessions | Multiple sessions simultaneously | ≥ 10 | Create 10 sessions, verify all active |

### Streaming Pipeline

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| Frame latency | Time from render to client display | ≤ 50 ms | Instrumented latency measurement |
| Frame rate | Consistent frame delivery | ≥ 30 FPS | Client-side FPS counter |
| Audio latency | Audio delay from engine to client | ≤ 100 ms | Audio-visual sync test |
| Input latency | Time from input to visual response | ≤ 100 ms | Input-to-frame measurement |
| Stream quality | Video encoding quality | PSNR ≥ 30 dB | Compare encoded vs. source frames |
| Connection stability | WebSocket connection uptime | ≥ 99.9% per hour | Monitor disconnection events |

### Azure Deployment

| Criterion | Metric | Threshold | Measurement |
|-----------|--------|-----------|-------------|
| IaC deployment | `az deployment` succeeds | Pass/Fail | Deployment command exits 0 |
| IaC idempotency | Repeated deployment has no changes | Pass/Fail | Run deploy twice, second shows no changes |
| Auto-scaling | Scales up under load | Scale to ≥ 3 pods | Generate load, observe HPA |
| Auto-scale down | Scales down when idle | Scale to 1 pod | Remove load, observe HPA |
| Public accessibility | Service reachable from internet | Pass/Fail | `curl https://quake.example.com` |
| TLS termination | HTTPS works | Valid certificate | Browser shows green lock |
| Monitoring | Azure Monitor receives metrics | Pass/Fail | Azure Portal shows data |
| Alerting | Alerts fire on threshold | Pass/Fail | Trigger alert condition, verify notification |

---

## 7. Overall Success KPIs

| KPI | Target | Measurement Frequency |
|-----|--------|----------------------|
| Security scan score | Zero critical/high findings | Every PR |
| Test pass rate | 100% | Every PR |
| Build success rate | > 99% | Monthly |
| Deployment success rate | > 95% | Monthly |
| Mean time to recovery | < 15 minutes | Per incident |
| Frame latency (P99) | < 50ms | Continuous |
| Session availability | > 99.9% | Monthly |
| Cost per session-hour | < $0.50 | Monthly |
