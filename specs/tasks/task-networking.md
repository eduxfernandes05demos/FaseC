# Task: Networking Subsystem

> Technical implementation specification with file references.

---

## Subsystem Overview

The networking subsystem provides two distinct implementations: WinQuake's LAN-focused reliable datagram protocol and QuakeWorld's internet-optimized delta compression system.

---

## WinQuake Networking Files (`legacy-src/desktop-engine/`)

| File | Responsibility |
|------|----------------|
| `net_main.c` | Network manager: initialization, driver dispatch, socket management |
| `net_dgrm.c` | Reliable datagram protocol: sequence numbers, ACK retransmission |
| `net_loop.c` | Loopback driver for single-machine play |
| `net_bsd.c` | BSD sockets infrastructure |
| `net_udp.c` | UDP socket implementation (Linux/BSD) |
| `net_wins.c` | Windows Winsock UDP |
| `net_wipx.c` | Windows IPX/SPX protocol |
| `net_ipx.c` | DOS IPX protocol |
| `net_ser.c` | Serial modem point-to-point |
| `net_comx.c` | COM port low-level access |
| `net_mp.c` / `net_mp.h` | Multi-player networking helpers |
| `net_bw.c` / `net_bw.h` | Bandwidth tracking |
| `net_vcr.c` / `net_vcr.h` | Network VCR recording (debug) |
| `net_none.c` | Null network driver |
| `net_dos.c` | DOS network initialization |
| `net_win.c` | Windows network initialization |
| `net.h` | Network interface, `qsocket_t` structure |
| `protocol.h` | Game protocol: message types, entity update flags |

---

## QuakeWorld Networking Files

### Server (`legacy-src/QW/server/`)
| File | Responsibility |
|------|----------------|
| `sv_main.c` | Server initialization, client management, main loop |
| `sv_send.c` | Message sending, PVS-based entity filtering, multicast |
| `sv_ents.c` | Delta compression encoding: `SV_WriteDelta()`, `SV_EmitPacketEntities()` |
| `sv_user.c` | User command processing, spectator handling |
| `sv_ccmds.c` | Server console commands (status, kick, map, master) |
| `sv_init.c` | Server spawn, level loading |
| `sv_nchan.c` | Reliable channel management, fragmentation |

### Client (`legacy-src/QW/client/`)
| File | Responsibility |
|------|----------------|
| `cl_main.c` | Client main loop, connection management |
| `cl_parse.c` | Server message parsing (53 svc_* message types) |
| `cl_ents.c` | Delta decompression: `CL_ParseDelta()`, `CL_ParsePacketEntities()` |
| `cl_pred.c` | Client-side prediction: `CL_PredictMove()`, `CL_PredictUsercmd()` |
| `cl_input.c` | Input command generation and sending |
| `cl_cam.c` | Chase camera (spectator view tracking) |
| `net_chan.c` | Network channel: reliable + unreliable dual stream |

### Shared
| File | Responsibility |
|------|----------------|
| `protocol.h` | QW protocol v28: messages, delta flags, constants |

---

## Key Data Structures

### WinQuake `qsocket_t` (`legacy-src/desktop-engine/net.h`)
- Connection state (active, send/receive capability)
- Reliable message buffer with sequence tracking
- ACK-based retransmission
- Per-driver data pointer

### QuakeWorld `client_t` (`legacy-src/QW/server/server.h`)
- `spectator` flag, `userid`, `userinfo` string
- Datagram buffer (unreliable), backup buffer (reliable)
- `frames[UPDATE_BACKUP]` — 64-frame circular buffer for delta reference
- `delta_sequence` — Client's acknowledged sequence for delta compression
- `chokecount` — Packet choke tracking
- `netchan_t` — Network channel with reliable/unreliable streams

### QuakeWorld `frame_t` (`legacy-src/QW/client/client.h`)
- `cmd` — usercmd_t that generated frame
- `senttime` / `receivedtime` — For latency calculation
- `delta_sequence` — Sequence to delta from
- `playerstate[]` — All players' states
- `packet_entities` — Entity states

---

## Protocol Summary

### WinQuake Protocol
- Full entity state snapshots each frame
- Reliable datagram with sequence numbers
- 4 client-to-server message types
- ~30 server-to-client message types

### QuakeWorld Protocol (v28)
- Delta-compressed entity updates
- 7 client-to-server message types
- 53 server-to-client message types
- PVS-based entity culling (only send visible entities)
- Master server registration

---

## Delta Compression Algorithm

### Server Side (`SV_WriteDelta()` — `legacy-src/QW/server/sv_ents.c:155`)
```
For each entity field (origin, angles, model, frame, etc.):
  if field changed since last acknowledged state:
    set corresponding flag bit
Write: entity_number | flags
Write: only changed field values
```

### Client Side (`CL_ParseDelta()` — `legacy-src/QW/client/cl_ents.c:160`)
```
Copy previous state → new state
Read: entity_number | flags
For each set flag:
  Read and overwrite that field
```

### Entity List Delta (`SV_EmitPacketEntities()` — `legacy-src/QW/server/sv_ents.c:250`)
```
Merge old and new entity lists:
  New entity (newnum < oldnum): Delta from baseline
  Same entity (newnum == oldnum): Delta from previous
  Removed entity (newnum > oldnum): U_REMOVE flag
```
