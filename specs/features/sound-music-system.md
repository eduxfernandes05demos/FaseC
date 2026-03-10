# Feature: Sound and Music System

> Reverse-engineered from `Quake/WinQuake/snd_*.c` and `Quake/WinQuake/cd_*.c`

---

## Overview

The Quake sound system provides **DMA-based audio** with **3D spatialization**, **multi-channel mixing**, and **CD audio music**. It supports multiple platform backends and renders audio asynchronously through a circular DMA buffer.

---

## Key Source Files

### Sound Engine
| File | Purpose |
|------|---------|
| `Quake/WinQuake/snd_dma.c` | Main sound system: initialization, channel management, 3D spatialization |
| `Quake/WinQuake/snd_mix.c` | Audio mixing engine — combines all channels into DMA buffer |
| `Quake/WinQuake/snd_mem.c` | WAV file loading and sample memory management |
| `Quake/WinQuake/sound.h` | Sound data structures: `channel_t`, `sfx_t`, `dma_t` |

### Platform Backends
| File | Platform | API |
|------|----------|-----|
| `Quake/WinQuake/snd_win.c` | Windows | DirectSound / waveOut |
| `Quake/WinQuake/snd_linux.c` | Linux | OSS (`/dev/dsp`) |
| `Quake/WinQuake/snd_sun.c` | Solaris | Sun audio device |
| `Quake/WinQuake/snd_dos.c` | DOS | DMA controller |
| `Quake/WinQuake/snd_gus.c` | DOS | Gravis UltraSound |
| `Quake/WinQuake/snd_null.c` | Any | Null driver (stub) |
| `Quake/WinQuake/snd_next.c` | NeXTSTEP | NeXT audio |

### CD Audio
| File | Platform |
|------|----------|
| `Quake/WinQuake/cd_audio.c` | Shared CD audio logic |
| `Quake/WinQuake/cd_win.c` | Windows MCI |
| `Quake/WinQuake/cd_linux.c` | Linux CD-ROM ioctl |
| `Quake/WinQuake/cd_null.c` | Null driver |

### Assembly Optimizations
| File | Purpose |
|------|---------|
| `Quake/WinQuake/snd_mixa.s` | x86 assembly mixing loop |

---

## Architecture

### DMA Circular Buffer

```
┌─────────────────────────────────────┐
│         DMA Buffer (circular)        │
│  ┌──────────────────────────────┐   │
│  │ mixed │ mixed │ being │ free │   │
│  │ audio │ audio │ mixed │      │   │
│  └──────────────────────────────┘   │
│     ↑ samplepos    ↑ paintedtime    │
│     (hw read)      (sw write)       │
└─────────────────────────────────────┘
```

- Hardware reads from `samplepos` asynchronously
- Software writes ("paints") starting from `paintedtime`
- Mixer fills the gap between hardware position and painted position

### Data Structures

**`dma_t`** (DMA buffer descriptor — `Quake/WinQuake/sound.h`):
| Field | Purpose |
|-------|---------|
| `channels` | Mono (1) or stereo (2) |
| `samples` | Total buffer size in samples |
| `samplepos` | Current hardware read position |
| `samplebits` | 8-bit or 16-bit samples |
| `speed` | Sample rate: 11025, 22050, or 44100 Hz |
| `buffer` | Pointer to circular buffer memory |

**`channel_t`** (playing sound — `Quake/WinQuake/sound.h`):
| Field | Purpose |
|-------|---------|
| `sfx` | Sound effect handle |
| `leftvol`, `rightvol` | Per-ear volume (0–255) |
| `end` | Sample time when sound ends |
| `pos` | Current playback position in sample |
| `looping` | Loop start position (-1 = no loop) |
| `entnum` | Entity that owns this sound |
| `origin` | 3D world position |
| `dist_mult` | Distance attenuation multiplier |

**`sfx_t`** (sound effect — `Quake/WinQuake/sound.h`):
| Field | Purpose |
|-------|---------|
| `name` | Sound file name |
| `cache` | Cached sample data pointer |

---

## Sound Pipeline

### Per-Frame Update (`S_Update()` in `Quake/WinQuake/snd_dma.c`)

1. Update listener position (from player view)
2. For each active channel:
   a. Calculate 3D direction from listener to sound source
   b. `SND_Spatialize()` — compute left/right volumes based on:
      - Distance attenuation (`dist_mult`)
      - Stereo panning (dot product with listener right vector)
      - Master volume and per-channel volume
   c. Kill channel if volume reaches zero (too far away)
3. Mix all active channels into DMA buffer (`S_PaintChannels()` in `snd_mix.c`)
4. Transfer mixed samples to hardware buffer

### 3D Spatialization (`SND_Spatialize()` in `Quake/WinQuake/snd_dma.c`)

```
1. Calculate vector from listener to sound source
2. Compute distance
3. Apply distance attenuation: volume = master_volume / (1.0 + distance * dist_mult)
4. Project onto listener's right vector for stereo panning:
   - right_scale = 1.0 + dot(direction, right_vector)
   - left_scale  = 1.0 - dot(direction, right_vector)
5. leftvol  = volume * left_scale
6. rightvol = volume * right_scale
```

### Mixing (`S_PaintChannels()` in `Quake/WinQuake/snd_mix.c`)

1. Zero-fill paint buffer for frame
2. For each channel, mix samples weighted by left/right volume
3. Clip output to valid range (8-bit: 0–255, 16-bit: -32768–32767)
4. Write to DMA buffer at `paintedtime` position

---

## Sound Channels

| Channel | Usage |
|---------|-------|
| 0 | Auto-assign |
| 1 | Weapon sounds |
| 2 | Voice/pain sounds |
| 3 | Item pickup sounds |
| 4+ | Additional channels |

**Attenuation constants:**
| Constant | Value | Usage |
|----------|-------|-------|
| `ATTN_NONE` | 0 | No distance falloff (global) |
| `ATTN_NORM` | 1 | Normal falloff |
| `ATTN_IDLE` | 2 | Faster falloff (ambient) |
| `ATTN_STATIC` | 3 | Very fast falloff (point source) |

---

## Configuration CVars

| CVar | Default | Purpose |
|------|---------|---------|
| `volume` | 0.7 | Master sound volume |
| `bgmvolume` | 1.0 | CD music volume |
| `nosound` | 0 | Disable sound |
| `ambient_level` | 0.3 | Ambient sound volume |
| `ambient_fade` | 100 | Ambient fade rate |
| `snd_noextraupdate` | 0 | Skip extra mixing updates |

---

## CD Audio

The CD audio system provides background music by playing audio CDs.

**Interface** (`Quake/WinQuake/cdaudio.h`):
- `CDAudio_Init()` — Initialize CD drive
- `CDAudio_Play(byte track, qboolean looping)` — Play CD track
- `CDAudio_Stop()` — Stop playback
- `CDAudio_Pause()` / `CDAudio_Resume()` — Pause/resume
- `CDAudio_Update()` — Check for track completion, handle looping

---

## Limitations

- Mono or stereo only (no surround sound)
- No audio effects (reverb, echo, EQ)
- No streaming (all sounds loaded fully into memory)
- 8-bit or 16-bit only (no float audio)
- Sample rates limited to 11025/22050/44100 Hz
- No HRTF or advanced 3D audio
- CD audio requires physical CD-ROM drive
- OSS-only on Linux (no ALSA, PulseAudio, PipeWire)
