# Task: Integration Tests — Service Boundaries

> Integration tests for service boundaries: session manager, gateway, engine container.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P1 — Testing |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 1 week |
| **Dependencies** | Container system, Session manager, Streaming gateway |
| **Blocks** | E2E testing, Production readiness |
| **Phase** | Phase 3-4 — Cloud-Ready / Cloud-Native (Testing) |

---

## Objective

Validate the interactions between service boundaries — engine container, session manager, and streaming gateway — ensuring correct communication, error handling, and resource management at each integration point.

---

## Integration Boundaries

```
┌─────────────────┐
│ Session Manager  │
│                  │
│  ┌────────┐     │     Boundary 1: Session Manager ↔ Kubernetes
│  │ REST   │─────│────▶ K8s API (Pod CRUD)
│  │ API    │     │
│  └────────┘     │     Boundary 2: Session Manager ↔ Engine Health
│                  │────▶ Engine /healthz, /readyz
└─────────────────┘
         │
         │ Boundary 3: Session Manager ↔ Gateway
         ▼
┌─────────────────┐
│ Streaming        │
│ Gateway          │     Boundary 4: Gateway ↔ Engine Framebuffer
│                  │────▶ VID_GetFramebuffer() [vid_headless.c]
│  ┌────────┐     │
│  │WebSocket│    │     Boundary 5: Gateway ↔ Engine Audio
│  │ Server │     │────▶ SND_GetCapturedAudio() [snd_null.c]
│  └────────┘     │
│                  │     Boundary 6: Gateway ↔ Engine Input
│                  │────▶ IN_InjectRemoteInput() [in_null.c]
└─────────────────┘
         │
         │ Boundary 7: Gateway ↔ Client (WebSocket)
         ▼
┌─────────────────┐
│ Browser Client   │
└─────────────────┘
```

---

## Test Suite: Boundary 1 — Session Manager ↔ Kubernetes

### INT-1.1: Pod Creation

| Test | Description | Expected |
|------|-------------|----------|
| `test_create_pod` | Session Manager creates engine pod via K8s API | Pod enters Running state |
| `test_create_pod_config` | Pod has correct env vars, ports, resources | Env vars match request |
| `test_create_pod_labels` | Pod has session labels for routing | Labels present |
| `test_create_pod_security` | Pod runs as non-root with security context | SecurityContext applied |

### INT-1.2: Pod Lifecycle

| Test | Description | Expected |
|------|-------------|----------|
| `test_delete_pod` | Session Manager deletes pod gracefully | Pod enters Terminating → gone |
| `test_pod_crash_detection` | Detect pod CrashLoopBackOff | Session marked as failed |
| `test_pod_oom_detection` | Detect OOMKilled pod | Session cleaned up |

---

## Test Suite: Boundary 2 — Session Manager ↔ Engine Health

### INT-2.1: Health Probing

| Test | Description | Expected |
|------|-------------|----------|
| `test_liveness_probe` | Engine `/healthz` responds 200 | Session marked alive |
| `test_readiness_probe` | Engine `/readyz` responds 200 after init | Session marked ready |
| `test_readiness_during_init` | Engine `/readyz` responds 503 during startup | Session not routed to |
| `test_health_failure` | Engine `/healthz` stops responding | Session marked unhealthy |

### INT-2.2: Health Data

| Test | Description | Expected |
|------|-------------|----------|
| `test_health_metrics` | `/healthz` includes basic metrics | JSON with uptime, players |
| `test_health_concurrent` | Multiple rapid health checks | All respond correctly |

---

## Test Suite: Boundary 3 — Session Manager ↔ Gateway

### INT-3.1: Session Routing

| Test | Description | Expected |
|------|-------------|----------|
| `test_route_to_session` | Gateway gets session endpoint from Manager | Correct IP:port returned |
| `test_route_invalid_session` | Gateway requests non-existent session | 404 response |
| `test_route_stopped_session` | Gateway requests stopped session | 410 Gone response |

---

## Test Suite: Boundary 4 — Gateway ↔ Engine Framebuffer

### INT-4.1: Frame Extraction

| Test | Description | Expected |
|------|-------------|----------|
| `test_framebuffer_read` | Read framebuffer after render | Valid pixel data returned |
| `test_framebuffer_dimensions` | Width/height match configuration | 640×480 default |
| `test_framebuffer_content` | Frame has non-zero pixels | Not all black (rendering works) |
| `test_framebuffer_timing` | Extraction takes < 2ms | Performance within budget |

### INT-4.2: Frame Consistency

| Test | Description | Expected |
|------|-------------|----------|
| `test_frame_sequence` | 100 sequential frames extracted | All valid, no corruption |
| `test_frame_after_map_load` | Frame extracted after map change | New map visible |
| `test_frame_during_gameplay` | Frames during active play | Entities move between frames |

**Key source references**:
- Rendering: `legacy-src/desktop-engine/r_main.c:1049` — `R_RenderView()`
- Screen update: `legacy-src/desktop-engine/screen.c` — `SCR_UpdateScreen()`
- Framebuffer: `vid_headless.c` — `VID_GetFramebuffer()`

---

## Test Suite: Boundary 5 — Gateway ↔ Engine Audio

### INT-5.1: Audio Capture

| Test | Description | Expected |
|------|-------------|----------|
| `test_audio_capture_init` | Audio capture initializes | Buffer allocated |
| `test_audio_samples` | Read captured audio samples | Non-zero PCM data |
| `test_audio_format` | Sample rate and channels correct | 44100 Hz, stereo |
| `test_audio_timing` | Capture buffer fills in real-time | ~44100 samples/second |

**Key source references**:
- Sound mixing: `legacy-src/desktop-engine/snd_mix.c:261` — `S_PaintChannels()`
- 8-bit mixing: `legacy-src/desktop-engine/snd_mix.c:346` — `SND_PaintChannelFrom8()`
- 16-bit mixing: `legacy-src/desktop-engine/snd_mix.c:375` — `SND_PaintChannelFrom16()`
- Capture: `snd_null.c` — `SND_GetCapturedAudio()`

---

## Test Suite: Boundary 6 — Gateway ↔ Engine Input

### INT-6.1: Input Injection

| Test | Description | Expected |
|------|-------------|----------|
| `test_inject_forward` | Inject forward movement | Player moves forward |
| `test_inject_rotation` | Inject view angle change | View rotates |
| `test_inject_attack` | Inject attack button | Weapon fires |
| `test_inject_rapid` | Rapid input injection (60/sec) | All inputs processed |

### INT-6.2: Input Validation

| Test | Description | Expected |
|------|-------------|----------|
| `test_inject_bounds` | Extreme movement values | Clamped to valid range |
| `test_inject_null` | No input injected | Player stationary |
| `test_inject_concurrent` | Multiple inputs same frame | Last input wins |

**Key source references**:
- Input processing: `legacy-src/desktop-engine/cl_input.c` — `CL_SendMove()`
- Input injection: `in_null.c` — `IN_InjectRemoteInput()`
- Host frame: `legacy-src/desktop-engine/host.c:729` — `Host_Frame()`

---

## Test Suite: Boundary 7 — Gateway ↔ Client (WebSocket)

### INT-7.1: Connection Management

| Test | Description | Expected |
|------|-------------|----------|
| `test_ws_connect` | Client opens WebSocket | Connection accepted |
| `test_ws_disconnect` | Client closes WebSocket | Resources freed |
| `test_ws_invalid_origin` | Wrong origin header | Connection rejected |
| `test_ws_max_connections` | Exceed max connections | New connections rejected |

### INT-7.2: Protocol

| Test | Description | Expected |
|------|-------------|----------|
| `test_ws_video_frame` | Client receives video frame | Valid binary message |
| `test_ws_audio_frame` | Client receives audio frame | Valid binary message |
| `test_ws_input_message` | Client sends input JSON | Parsed without error |
| `test_ws_invalid_message` | Client sends garbage | Connection not crashed |
| `test_ws_rate_limit` | Client sends too fast | Rate limited |

---

## Test Infrastructure

### Docker Compose for Integration Testing

```yaml
# tests/integration/docker-compose.test.yml
version: '3.8'
services:
  engine:
    image: quake-engine:test
    environment:
      - QUAKE_PORT=26000
      - QUAKE_GAMEDIR=id1
      - QUAKE_MEMORY=32
      - HEALTH_PORT=8080

  session-manager:
    image: quake-session-mgr:test
    environment:
      - ENGINE_IMAGE=quake-engine:test
    depends_on:
      - engine

  gateway:
    image: quake-gateway:test
    ports:
      - "8080:8080"
    depends_on:
      - engine

  test-runner:
    image: quake-integration-tests:latest
    depends_on:
      - session-manager
      - gateway
    command: pytest tests/integration/ -v
```

---

## Acceptance Criteria

- [ ] All Boundary 1-7 tests pass
- [ ] No resource leaks detected during test runs
- [ ] All error paths tested (invalid input, service failure, timeout)
- [ ] Tests run in Docker Compose environment
- [ ] Tests complete within 10 minutes
- [ ] Tests are reproducible (no flaky tests)
- [ ] Tests integrated into CI pipeline
- [ ] Test coverage includes both happy path and error scenarios
