# Task: Feature Validation — Streaming End-to-End

> End-to-end validation of the streaming pipeline: client to cloud to rendered frame.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Validation |
| **Estimated Complexity** | High |
| **Estimated Effort** | 1.5 weeks |
| **Dependencies** | Streaming gateway, Session manager, Headless video, Container system |
| **Blocks** | Production readiness sign-off |
| **Phase** | Phase 4 — Cloud-Native (Testing) |

---

## Objective

Validate the complete streaming pipeline from browser client connection through cloud infrastructure to rendered game frames, ensuring end-to-end functionality, latency requirements, and user experience quality.

---

## Test Scenarios

### E2E-1: Full Connection Lifecycle

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Browser connects to gateway WebSocket | Connection established, session ID returned |
| 2 | Gateway requests new session from Session Manager | Engine container starts |
| 3 | Engine initializes in headless mode | `/readyz` returns 200, engine loads map |
| 4 | Gateway captures first frame | Non-black frame extracted from `VID_GetFramebuffer()` |
| 5 | Gateway encodes and streams frame | Client receives valid H.264 frame |
| 6 | Client sends input via WebSocket | Input relayed via `IN_InjectRemoteInput()` |
| 7 | Engine processes input | Player moves, next frame reflects movement |
| 8 | Client disconnects | Session cleaned up within 60 seconds |

**Files involved**:
- Engine rendering: `legacy-src/desktop-engine/r_main.c:1049` (`R_RenderView`)
- Frame extraction: `vid_headless.c` (`VID_GetFramebuffer()`)
- Sound capture: `snd_null.c` (`SND_GetCapturedAudio()`)
- Input injection: `in_null.c` (`IN_InjectRemoteInput()`)
- Screen update: `legacy-src/desktop-engine/screen.c` (`SCR_UpdateScreen()`)

### E2E-2: Multi-Session Concurrent Streaming

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Create 5 simultaneous sessions | All 5 engine containers running |
| 2 | Connect 5 browser clients | All receive video streams |
| 3 | Send input from all 5 clients | Each session responds independently |
| 4 | Measure latency for all sessions | All ≤ 50ms end-to-end |
| 5 | Disconnect 3 clients | 3 sessions cleaned up, 2 continue |
| 6 | Create 3 more sessions | Total 5 sessions again, all functional |

### E2E-3: Network Resilience

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Connect and start streaming | Normal operation |
| 2 | Simulate network interruption (5s) | Client detects disconnect |
| 3 | Restore network | Client reconnects automatically |
| 4 | Verify session preserved | Same game state, no restart needed |
| 5 | Simulate high packet loss (20%) | Adaptive quality reduces resolution/FPS |
| 6 | Restore network quality | Quality returns to normal |

### E2E-4: Audio-Video Synchronization

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Start streaming session | Video and audio streams active |
| 2 | Trigger in-game event (weapon fire) | Sound and visual effect occur simultaneously |
| 3 | Measure A/V sync offset | Offset ≤ 50ms |
| 4 | Stream for 10 minutes | A/V sync maintained (no drift) |

### E2E-5: Long-Running Stability

| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | Start streaming session | Normal operation |
| 2 | Stream continuously for 4 hours | No memory leaks, no degradation |
| 3 | Monitor memory usage | Stays within container limits |
| 4 | Monitor frame rate | Consistent 30+ FPS throughout |
| 5 | Monitor latency | Consistent ≤ 50ms throughout |

---

## Test Implementation

### Test Framework

```python
# pytest-based E2E test suite
# File: tests/e2e/test_streaming_e2e.py

import pytest
import websocket
import json
import time

@pytest.fixture
def session_manager_url():
    return "http://localhost:8090"

@pytest.fixture
def gateway_url():
    return "ws://localhost:8080"

class TestStreamingE2E:
    def test_full_connection_lifecycle(self, gateway_url, session_manager_url):
        """E2E-1: Full connection lifecycle"""
        # Create session
        resp = requests.post(f"{session_manager_url}/api/sessions",
                           json={"map": "start", "maxPlayers": 8})
        assert resp.status_code == 201
        session = resp.json()

        # Wait for session to be ready
        wait_for_ready(session["healthUrl"], timeout=30)

        # Connect WebSocket
        ws = websocket.create_connection(
            f"{gateway_url}/stream/{session['id']}")

        # Receive first video frame
        frame = ws.recv()
        assert len(frame) > 0
        assert frame[0] == 0x01  # Video frame type

        # Send input
        ws.send(json.dumps({
            "type": "input",
            "forward": 400.0,
            "yaw": 45.0,
            "buttons": 0
        }))

        # Receive updated frame
        time.sleep(0.1)
        frame2 = ws.recv()
        assert len(frame2) > 0

        # Disconnect
        ws.close()

        # Verify cleanup
        time.sleep(70)
        resp = requests.get(
            f"{session_manager_url}/api/sessions/{session['id']}")
        assert resp.status_code == 404  # Session cleaned up

    def test_multi_session(self, gateway_url, session_manager_url):
        """E2E-2: Multiple concurrent sessions"""
        sessions = []
        for i in range(5):
            resp = requests.post(f"{session_manager_url}/api/sessions",
                               json={"map": "start"})
            assert resp.status_code == 201
            sessions.append(resp.json())

        # Verify all running
        for session in sessions:
            wait_for_ready(session["healthUrl"])
            ws = websocket.create_connection(
                f"{gateway_url}/stream/{session['id']}")
            frame = ws.recv()
            assert len(frame) > 0
            ws.close()

    def test_latency(self, gateway_url, session_manager_url):
        """Verify end-to-end latency"""
        # Create session and connect
        session = create_session(session_manager_url)
        ws = websocket.create_connection(
            f"{gateway_url}/stream/{session['id']}")

        latencies = []
        for _ in range(100):
            start = time.monotonic()
            ws.send(json.dumps({"type": "input", "forward": 400.0}))
            frame = ws.recv()
            latency = (time.monotonic() - start) * 1000
            latencies.append(latency)

        p50 = sorted(latencies)[50]
        p95 = sorted(latencies)[95]
        p99 = sorted(latencies)[99]

        assert p50 <= 30, f"P50 latency {p50}ms exceeds 30ms"
        assert p95 <= 50, f"P95 latency {p95}ms exceeds 50ms"
        assert p99 <= 100, f"P99 latency {p99}ms exceeds 100ms"

        ws.close()
```

---

## Measurement Points

| Measurement | Source | Method |
|-------------|--------|--------|
| Session creation time | Session Manager | API response time |
| Engine startup time | Health endpoint | Time to `/readyz` = 200 |
| First frame latency | Gateway | Time from connection to first frame |
| Frame-to-frame latency | Client | Input → visual response time |
| Audio latency | Client | A/V sync measurement |
| Memory usage trend | Container | `/proc/pid/status` VmRSS |
| Frame rate stability | Client | FPS counter variance |

---

## Acceptance Criteria

- [ ] E2E-1: Full lifecycle completes without errors
- [ ] E2E-2: 5 concurrent sessions stream simultaneously
- [ ] E2E-3: Client reconnects after network interruption
- [ ] E2E-4: Audio-video sync within 50ms
- [ ] E2E-5: 4-hour stability test passes (no leaks, no degradation)
- [ ] P95 end-to-end latency ≤ 50ms
- [ ] Frame rate ≥ 30 FPS consistently
- [ ] Session cleanup within 60 seconds of disconnect
- [ ] All tests automated and runnable in CI
