# Feature: Client-Side Prediction

> Reverse-engineered from `Quake/QW/client/cl_pred.c` and `Quake/QW/client/cl_ents.c`

---

## Overview

Client-side prediction is a key QuakeWorld innovation that masks network latency by running player physics locally on the client. The player sees their movement immediately rather than waiting for a server round-trip, creating a responsive gameplay feel even on high-latency internet connections.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/QW/client/cl_pred.c` | Prediction algorithm — `CL_PredictMove()`, `CL_PredictUsercmd()` |
| `Quake/QW/client/cl_ents.c` | Entity state management, `CL_SetUpPlayerPrediction()` |
| `Quake/QW/client/cl_input.c` | Input command generation (`usercmd_t`) |
| `Quake/QW/client/client.h` | `frame_t`, `player_state_t` structures |
| `Quake/QW/client/protocol.h` | `usercmd_t` structure |

---

## Algorithm

### 1. Input Sampling
- Each frame, client creates a `usercmd_t` with movement intent (forward/side/up, angles, buttons, timestamp)
- Command is sent to server and stored locally in frame history

### 2. Server Acknowledgment
- Server processes commands and sends back authoritative state
- `delta_sequence` in packets tells client which frame the server last processed

### 3. Prediction (`CL_PredictMove()` — line ~112)
```
1. Find last server-acknowledged frame (delta_sequence)
2. Start from server-confirmed position
3. For each unacknowledged command since then:
   a. Call CL_PredictUsercmd() — simulate one command
   b. Split moves > 50ms into smaller chunks
4. Interpolate to current time
5. Result: predicted position for rendering
```

### 4. Single-Command Prediction (`CL_PredictUsercmd()` — line ~64)
- Applies user input to local physics simulation
- Handles: gravity, friction, acceleration, ground detection
- Uses player collision hull against BSP world
- Produces new predicted origin and velocity

### 5. Reconciliation
- When server state arrives, compare prediction with reality
- If position differs: snap to server position (visible "correction")
- Rerun unacknowledged commands from corrected position

---

## Data Flow

```
Frame N: Client sends usercmd → Server receives
Frame N+1: Client sends usercmd → Server processes Frame N
Frame N+2: Client sends usercmd → Server sends back state for Frame N
                                    Client: Compare prediction vs server
                                    Repredict from N using N+1, N+2 inputs
```

---

## Key Data Structures

### `frame_t` (client.h)
```
cmd              — usercmd_t that generated this frame
senttime         — Time command was sent
delta_sequence   — Sequence number to delta from (-1 = full)
receivedtime     — Time server state was received
playerstate[]    — Per-player state array
packet_entities  — Entity states in this frame
invalid          — True if delta reference was lost
```

### `player_state_t` (client.h)
```
messagenum    — Server message number
state_time    — Server time for this state
command       — Last usercmd for replay
origin        — Position
viewangles    — View direction
velocity      — Movement speed
weaponframe   — Weapon animation frame
modelindex    — Player model
flags         — Dead, gib, etc.
onground      — Ground entity (-1 = airborne)
```

Frame buffer: `UPDATE_BACKUP = 64` frames stored in circular buffer.

---

## Limitations

- **Prediction misprediction**: When prediction disagrees with server (e.g., collision with another player the client doesn't know about), the player "snaps" to the correct position
- **No prediction for other players**: Only the local player is predicted; other players interpolate between server updates
- **Platform physics differences**: Client and server must use identical physics code or predictions will always fail
- **No movement smoothing**: Corrections are instantaneous snaps, not smoothed over time
- **50ms minimum granularity**: Commands are split into 50ms chunks, which can cause slight inaccuracy
