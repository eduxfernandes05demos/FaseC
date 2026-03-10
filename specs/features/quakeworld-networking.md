# Feature: QuakeWorld Multiplayer Networking

> Reverse-engineered from `Quake/QW/client/` and `Quake/QW/server/`

---

## Overview

QuakeWorld implements an internet-optimized multiplayer networking system that addresses the high-latency limitations of WinQuake's LAN-focused networking. Key innovations include **delta compression** (reducing bandwidth ~90%), **client-side prediction** (masking latency), and a **dedicated server** architecture (separating server from client).

---

## Key Source Files

### Server
| File | Purpose |
|------|---------|
| `Quake/QW/server/sv_main.c` | Server initialization, main loop, client management |
| `Quake/QW/server/sv_send.c` | Message sending, PVS-based entity filtering |
| `Quake/QW/server/sv_ents.c` | Entity delta compression and encoding |
| `Quake/QW/server/sv_user.c` | User command processing, spectator handling |
| `Quake/QW/server/sv_ccmds.c` | Server console commands (status, kick, map, etc.) |
| `Quake/QW/server/sv_init.c` | Server spawn, level loading |
| `Quake/QW/server/sv_phys.c` | Server-side physics simulation |
| `Quake/QW/server/sv_move.c` | Entity movement and pathfinding |
| `Quake/QW/server/sv_nchan.c` | Reliable network channel management |

### Client
| File | Purpose |
|------|---------|
| `Quake/QW/client/cl_main.c` | Client main loop, connection management |
| `Quake/QW/client/cl_parse.c` | Server message parsing (svc_* handlers) |
| `Quake/QW/client/cl_ents.c` | Entity delta decompression, entity linking |
| `Quake/QW/client/cl_pred.c` | Client-side movement prediction |
| `Quake/QW/client/cl_input.c` | User input sampling and command generation |
| `Quake/QW/client/cl_demo.c` | Demo recording and playback |
| `Quake/QW/client/cl_cam.c` | Chase camera (spectator view) |

### Shared
| File | Purpose |
|------|---------|
| `Quake/QW/client/protocol.h` | Protocol constants, message types, delta flags |
| `Quake/QW/client/net_chan.c` | Network channel (reliable + unreliable streams) |

---

## Protocol Details

**Protocol version**: 28 (`Quake/QW/client/protocol.h`)

### Ports
| Port | Purpose |
|------|---------|
| 27500 | Game server |
| 27001 | Game client |
| 27000 | Master server |

### Server-to-Client Messages (53 types)

Key messages include:
| Message | ID | Purpose |
|---------|-----|---------|
| `svc_serverdata` | 11 | Initial connection data (protocol, map, etc.) |
| `svc_packetentities` | 47 | Full entity state packet (delta from baseline) |
| `svc_deltapacketentities` | 48 | Delta entity state packet (delta from previous) |
| `svc_playerinfo` | 42 | Player state with delta flags |
| `svc_updateping` | 36 | Client ping time |
| `svc_updatepl` | 53 | Packet loss percentage |
| `svc_download` | 41 | File download chunk |
| `svc_modellist` | 45 | Server model list |
| `svc_soundlist` | 46 | Server sound list |
| `svc_maxspeed` | 49 | Client maximum speed |
| `svc_entgravity` | 50 | Client gravity scale |

### Client-to-Server Messages
| Message | ID | Purpose |
|---------|-----|---------|
| `clc_move` | 3 | User movement command (`usercmd_t`) |
| `clc_stringcmd` | 4 | Text command |
| `clc_delta` | 5 | Delta sequence acknowledgment |
| `clc_tmove` | 6 | Teleport move |
| `clc_upload` | 7 | File upload |

---

## Delta Compression

### Server-Side Encoding (`Quake/QW/server/sv_ents.c`)

**`SV_WriteDelta()`** (line ~155):
1. Compare source and destination `entity_state_t` fields
2. Set flag bits for each changed field
3. Write entity number + combined flags as 16-bit short
4. If `U_MOREBITS`: write extended flags byte
5. For each set flag: write only that field's data

**`SV_EmitPacketEntities()`** (line ~250):
1. Compare old entity list with new
2. For each entity:
   - `newnum < oldnum`: New entity → delta from baseline
   - `newnum == oldnum`: Existing entity → delta from previous
   - `newnum > oldnum`: Removed entity → `U_REMOVE` flag
3. Write `0x0000` to mark end of entity list

### Client-Side Decoding (`Quake/QW/client/cl_ents.c`)

**`CL_ParseDelta()`** (line ~160):
1. Copy previous entity state to destination
2. Extract entity number from low 9 bits
3. If `U_MOREBITS` set, read extended flags byte
4. For each flag set: read and update that field only
5. Unchanged fields remain from previous state

### Delta Flags (`Quake/QW/client/protocol.h`)

| Flag | Bit | Field | Encoding |
|------|-----|-------|----------|
| `U_ORIGIN1` | `1<<9` | X position | `MSG_WriteCoord` (16-bit, 1/8 unit) |
| `U_ORIGIN2` | `1<<10` | Y position | `MSG_WriteCoord` |
| `U_ORIGIN3` | `1<<11` | Z position | `MSG_WriteCoord` |
| `U_ANGLE2` | `1<<12` | Pitch angle | `MSG_WriteAngle` (byte) |
| `U_FRAME` | `1<<13` | Animation frame | byte |
| `U_REMOVE` | `1<<14` | Entity removed | — |
| `U_MOREBITS` | `1<<15` | Extended flags | byte |
| `U_ANGLE1` | `1<<0` (ext) | Yaw angle | byte |
| `U_ANGLE3` | `1<<1` (ext) | Roll angle | byte |
| `U_MODEL` | `1<<2` (ext) | Model index | byte |
| `U_COLORMAP` | `1<<3` (ext) | Color map | byte |
| `U_SKIN` | `1<<4` (ext) | Skin number | byte |
| `U_EFFECTS` | `1<<5` (ext) | Effects flags | byte |
| `U_SOLID` | `1<<6` (ext) | Solid flag | — |

---

## Client-Side Prediction (`Quake/QW/client/cl_pred.c`)

### Algorithm

**`CL_PredictMove()`** (line ~112):
1. Get last acknowledged server frame
2. For each unacknowledged user command since then:
   - Run `CL_PredictUsercmd()` to simulate movement
   - Split large time steps into 50ms chunks
3. Interpolate final position based on fractional time
4. Result: predicted player position rendered with no perceived latency

**`CL_PredictUsercmd()`** (line ~64):
- Applies user movement input to local physics simulation
- Uses player physics hull for collision detection
- Handles ground detection, gravity, friction, acceleration

### Key Data Structures

**`frame_t`** (`Quake/QW/client/client.h`):
```
- cmd: usercmd_t         — Command that generated this frame
- senttime: double       — Time command was sent
- delta_sequence: int    — Sequence to delta from (-1 = full)
- receivedtime: double   — Time server state was received
- playerstate[]          — Player state array
- packet_entities        — Entity state list
- invalid: qboolean     — True if delta was corrupted
```

Circular buffer of `UPDATE_BACKUP = 64` frames for prediction replay.

---

## Server Features

### Spectator Mode
- `client->spectator` flag in `client_t` (`Quake/QW/server/server.h:121`)
- Spectators receive entity updates but cannot interact with the game
- Can track specific player via `spec_track` field
- Separate password (`spectator_password` cvar)

### Anti-Cheat Measures
- **Model checksums**: `model_player_checksum`, `eyes_player_checksum` (`Quake/QW/server/server.h`)
- **Download restrictions**: `allow_download`, `allow_download_skins`, `allow_download_models`, `allow_download_sounds`, `allow_download_maps` cvars
- **Choke counting**: `chokecount` field tracks dropped packets
- **Delta sequence validation**: Ensures client isn't spoofing frame references
- **RCON password**: Remote console access protection

### Master Server Registration
- `SV_SetMaster_f()` in `Quake/QW/server/sv_ccmds.c` registers server with master servers
- Allows server discovery through master server queries

---

## QW Forwarding Proxy (`Quake/QW/qwfwd/`)

A transparent UDP relay that forwards traffic between clients and servers.

| File | Purpose |
|------|---------|
| `Quake/QW/qwfwd/qwfwd.c` | Main proxy implementation (269 lines) |
| `Quake/QW/qwfwd/misc.c` | Utility functions (452 lines) |

**Architecture**: Maintains `peer_t` linked list mapping client addresses to server connections. Uses `select()` for non-blocking I/O. Peers time out after 2 minutes of inactivity.

---

## Limitations

- No encryption — all data transmitted in plaintext
- No authentication beyond password strings
- UDP only — no TCP fallback for firewalls
- Maximum 32 players (`MAX_CLIENTS`)
- No anti-wallhack (PVS culling is server-side but limited)
- Prediction errors cause visible "snapping" corrections
