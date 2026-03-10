# Feature: Demo Recording and Playback

> Reverse-engineered from `Quake/WinQuake/cl_demo.c` and `Quake/QW/client/cl_demo.c`

---

## Overview

The demo system records and plays back gameplay sessions by capturing the stream of network messages between server and client. This allows replaying games for review, entertainment, or benchmarking.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/WinQuake/cl_demo.c` | WinQuake demo recording and playback |
| `Quake/QW/client/cl_demo.c` | QuakeWorld demo recording and playback |

---

## Demo Format

Demo files (`.dem`) contain a sequential stream of network messages with timing information:

### WinQuake Demo Format
```
[float time] [int size] [byte[] message_data]
[float time] [int size] [byte[] message_data]
...
```

Each block contains:
1. **Timestamp**: Server time when message was generated
2. **Message size**: Number of bytes in the message
3. **Message data**: Raw server-to-client message (same as network)

### QuakeWorld Demo Format
Similar structure with QuakeWorld protocol messages (delta-compressed entities, player info, etc.).

---

## Console Commands

| Command | Purpose |
|---------|---------|
| `record <name>` | Start recording to `<name>.dem` |
| `stop` | Stop recording |
| `playdemo <name>` | Play back a demo file |
| `timedemo <name>` | Play demo as fast as possible (benchmarking) |
| `startdemos <list>` | Queue demos for menu background playback |

---

## Key Functions

### Recording
- `CL_Record_f()` — Start recording: open file, write initial server info
- During gameplay: all incoming server messages are written to the demo file with timestamps
- `CL_Stop_f()` — Stop recording: close file

### Playback
- `CL_PlayDemo_f()` — Open demo file, set client into demo playback mode
- Each frame: read messages from demo file instead of network
- Messages are fed through the same `CL_ParseServerMessage()` path
- Timing: advance through messages based on recorded timestamps

### Time Demo (Benchmarking)
- `CL_TimeDemo_f()` — Play demo with no timing delays
- Renders every frame as fast as possible
- Reports total frames, elapsed time, and average FPS at completion
- Used as a standardized performance benchmark

---

## Limitations

- WinQuake demos not compatible with QuakeWorld demos (different protocols)
- No seeking or fast-forward (sequential playback only)
- No rewind capability
- Demo file size can be large for long sessions
- No camera control during playback (fixed to recorded viewpoint)
- Client-side effects may differ on playback (interpolation differences)
