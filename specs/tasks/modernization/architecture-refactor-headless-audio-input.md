# Task: Architecture Refactor — Headless Audio & Input

> Decouple audio/input from platform, enable headless mode without sound/input hardware.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P1 — Modularization |
| **Estimated Complexity** | Medium |
| **Estimated Effort** | 1 week |
| **Dependencies** | Build system (CMake), Security remediation |
| **Blocks** | Containerization |
| **Phase** | Phase 2 — Modularization |

---

## Objective

Enhance the existing null audio and input drivers to support headless operation with audio capture (for streaming) and remote input injection (for cloud gaming). The engine already has `snd_null.c` and `in_null.c` stubs — these need enhancement.

---

## Current Audio Architecture

| File | Platform | Key Functions |
|------|----------|---------------|
| `legacy-src/desktop-engine/snd_dma.c` | Core | `S_Init()`, `S_Update()`, `S_PaintChannels()` (line 261 in `snd_mix.c`) |
| `legacy-src/desktop-engine/snd_mix.c` | Core | `S_PaintChannels()` (line 261) — Main mixing loop |
| `legacy-src/desktop-engine/snd_mem.c` | Core | Sound loading |
| `legacy-src/desktop-engine/snd_win.c` | Windows | DirectSound output |
| `legacy-src/desktop-engine/snd_linux.c` | Linux | `/dev/dsp` output |
| `legacy-src/desktop-engine/snd_null.c` | Stub | **Existing** — empty stubs |

### Sound Mixing Pipeline

```
snd_mix.c:261 S_PaintChannels()     — Mix all active channels
  ├── snd_mix.c:346 SND_PaintChannelFrom8()  — 8-bit sample mixing (loop at 362)
  ├── snd_mix.c:375 SND_PaintChannelFrom16() — 16-bit sample mixing (loop at 387)
  └── snd_mix.c:65  S_TransferStereo16()     — Transfer to DMA buffer
```

### Current snd_null.c

The existing `snd_null.c` provides minimal stubs that prevent crashes but discard all audio:

```c
/* Current snd_null.c — all stubs return without doing anything */
qboolean SNDDMA_Init(void) { return 0; }
void SNDDMA_Shutdown(void) { }
void SNDDMA_Submit(void) { }
```

---

## Current Input Architecture

| File | Platform | Key Functions |
|------|----------|---------------|
| `legacy-src/desktop-engine/in_win.c` | Windows | `IN_Init()`, `IN_Move()`, `IN_Commands()` |
| `legacy-src/desktop-engine/in_dos.c` | DOS | `IN_Init()`, `IN_Move()` |
| `legacy-src/desktop-engine/in_sun.c` | Solaris | `IN_Init()`, `IN_Move()` |
| `legacy-src/desktop-engine/in_null.c` | Stub | **Existing** — empty stubs |
| `legacy-src/desktop-engine/keys.c` | Core | Key event processing |

### Current in_null.c

```c
/* Current in_null.c — all stubs */
void IN_Init(void) { }
void IN_Shutdown(void) { }
void IN_Commands(void) { }
void IN_Move(usercmd_t *cmd) { }
```

---

## Implementation Steps

### Step 1: Enhance snd_null.c for Audio Capture

**Modified file**: `legacy-src/desktop-engine/snd_null.c`

Add audio capture capability so the streaming pipeline can access mixed audio:

```c
/* Enhanced snd_null.c — Audio capture for streaming */
#include "quakedef.h"

static qboolean audio_capture_enabled = false;
static int16_t *capture_buffer = NULL;
static int capture_buffer_size = 0;
static int capture_write_pos = 0;

qboolean SNDDMA_Init(void) {
    /* In headless mode with streaming, enable capture */
    if (/* headless + streaming config */) {
        audio_capture_enabled = true;
        capture_buffer_size = 44100 * 2; /* 1 second stereo */
        capture_buffer = malloc(capture_buffer_size * sizeof(int16_t));
        if (!capture_buffer) return 0;
        /* Configure shm (DMA descriptor) for mixing */
        shm = &sn;
        shm->speed = 44100;
        shm->channels = 2;
        shm->samplebits = 16;
        shm->samples = capture_buffer_size;
        shm->buffer = (unsigned char *)capture_buffer;
        return 1;
    }
    return 0;  /* No audio in pure headless mode */
}

void SNDDMA_Shutdown(void) {
    if (capture_buffer) {
        free(capture_buffer);
        capture_buffer = NULL;
    }
}

void SNDDMA_Submit(void) {
    /* Buffer is already written by the mixer */
}

/* New: API for streaming pipeline to read captured audio */
int SND_GetCapturedAudio(int16_t *out, int max_samples) {
    if (!audio_capture_enabled || !capture_buffer)
        return 0;
    /* Copy available samples to output buffer */
    int available = capture_write_pos;
    int to_copy = (available < max_samples) ? available : max_samples;
    memcpy(out, capture_buffer, to_copy * sizeof(int16_t));
    capture_write_pos = 0;
    return to_copy;
}
```

### Step 2: Enhance in_null.c for Remote Input

**Modified file**: `legacy-src/desktop-engine/in_null.c`

Add remote input injection for cloud gaming:

```c
/* Enhanced in_null.c — Remote input injection */
#include "quakedef.h"

typedef struct {
    float forwardmove;
    float sidemove;
    float upmove;
    float viewangles[3];  /* pitch, yaw, roll */
    int buttons;          /* attack, jump, etc. */
    qboolean pending;
} remote_input_t;

static remote_input_t pending_input = {0};

void IN_Init(void) {
    /* No hardware initialization needed */
}

void IN_Shutdown(void) {
    memset(&pending_input, 0, sizeof(pending_input));
}

void IN_Commands(void) {
    /* Process pending button state into key events */
    if (pending_input.pending) {
        if (pending_input.buttons & 1) Key_Event(K_MOUSE1, true);
        if (pending_input.buttons & 2) Key_Event(K_MOUSE2, true);
        /* ... etc */
    }
}

void IN_Move(usercmd_t *cmd) {
    if (pending_input.pending) {
        cmd->forwardmove += pending_input.forwardmove;
        cmd->sidemove += pending_input.sidemove;
        cmd->upmove += pending_input.upmove;
        cl.viewangles[0] = pending_input.viewangles[0];
        cl.viewangles[1] = pending_input.viewangles[1];
        cl.viewangles[2] = pending_input.viewangles[2];
        pending_input.pending = false;
    }
}

/* New: API for streaming gateway to inject input */
void IN_InjectRemoteInput(float forward, float side, float up,
                           float pitch, float yaw, float roll,
                           int buttons) {
    pending_input.forwardmove = forward;
    pending_input.sidemove = side;
    pending_input.upmove = up;
    pending_input.viewangles[0] = pitch;
    pending_input.viewangles[1] = yaw;
    pending_input.viewangles[2] = roll;
    pending_input.buttons = buttons;
    pending_input.pending = true;
}
```

### Step 3: Enhance CD Audio Null Driver

**File**: `legacy-src/desktop-engine/cd_null.c` — Already a complete null implementation. No changes needed unless music capture is required for streaming.

---

## Files Affected

| File | Change Type | Description |
|------|------------|-------------|
| `legacy-src/desktop-engine/snd_null.c` | **Enhance** | Add audio capture capability |
| `legacy-src/desktop-engine/in_null.c` | **Enhance** | Add remote input injection |
| `legacy-src/desktop-engine/sound.h` | Minor edit | Add `SND_GetCapturedAudio()` declaration |
| `legacy-src/desktop-engine/input.h` | Minor edit | Add `IN_InjectRemoteInput()` declaration |
| `legacy-src/desktop-engine/CMakeLists.txt` | Edit | Include null drivers in headless target |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_snd_null_init` | `SNDDMA_Init()` succeeds in headless mode |
| `test_snd_capture` | Audio capture buffer receives mixed samples |
| `test_snd_capture_read` | `SND_GetCapturedAudio()` returns correct sample count |
| `test_in_null_init` | `IN_Init()` succeeds in headless mode |
| `test_in_inject` | `IN_InjectRemoteInput()` sets pending state |
| `test_in_move` | `IN_Move()` applies injected input to usercmd_t |
| `test_headless_full` | Engine runs headless with null audio and input without crashes |

---

## Acceptance Criteria

- [ ] Engine starts in headless mode without audio hardware
- [ ] Engine starts in headless mode without input hardware
- [ ] Sound mixing pipeline runs (channels mixed into capture buffer)
- [ ] `SND_GetCapturedAudio()` returns mixed audio samples
- [ ] `IN_InjectRemoteInput()` successfully injects player input
- [ ] Injected input moves the player in the game world
- [ ] No audio-related error messages or crashes in headless mode
- [ ] No input-related error messages or crashes in headless mode
- [ ] Memory usage of null audio/input drivers ≤ 1 MB combined
