# Task: Sound Subsystem

> Technical implementation specification with file references.

---

## Subsystem Overview

The sound subsystem provides DMA-based audio mixing with 3D spatialization, supporting multiple platform backends.

---

## File Inventory (`legacy-src/desktop-engine/`)

### Core Sound Engine
| File | Responsibility |
|------|----------------|
| `snd_dma.c` | Sound manager: initialization, channel allocation, 3D spatialization, update loop |
| `snd_mix.c` | Multi-channel audio mixing into DMA buffer |
| `snd_mem.c` | WAV file loading, sample caching |
| `sound.h` | Data structures: `dma_t`, `channel_t`, `sfx_t`, `sfxcache_t` |

### Platform Backends
| File | Platform | API |
|------|----------|-----|
| `snd_win.c` | Windows | DirectSound / waveOut |
| `snd_linux.c` | Linux | OSS `/dev/dsp` |
| `snd_sun.c` | Solaris | Sun audio device |
| `snd_dos.c` | DOS | DMA controller direct access |
| `snd_gus.c` | DOS | Gravis UltraSound card |
| `snd_next.c` | NeXTSTEP | NeXT audio system |
| `snd_null.c` | Any | Null/stub driver |

### CD Audio
| File | Platform |
|------|----------|
| `cd_audio.c` | Shared CD logic |
| `cd_win.c` | Windows MCI API |
| `cd_linux.c` | Linux ioctl |
| `cd_null.c` | Null driver |
| `cdaudio.h` | CD audio interface |

### Assembly
| File | Purpose |
|------|---------|
| `snd_mixa.s` | x86 optimized mixing inner loop |

---

## Key Functions

### Initialization
| Function | File | Purpose |
|----------|------|---------|
| `S_Init()` | `snd_dma.c` | Initialize sound system, register cvars |
| `S_Startup()` | `snd_dma.c` | Start DMA transfer |
| `S_Shutdown()` | `snd_dma.c` | Stop DMA, release resources |

### Sound Control
| Function | File | Purpose |
|----------|------|---------|
| `S_StartSound()` | `snd_dma.c` | Start 3D-positioned sound on channel |
| `S_StopSound()` | `snd_dma.c` | Stop sound on specific channel |
| `S_StaticSound()` | `snd_dma.c` | Start looping ambient sound |
| `S_StopAllSounds()` | `snd_dma.c` | Stop all channels |
| `S_PrecacheSound()` | `snd_dma.c` | Load and cache sound file |

### Per-Frame Update
| Function | File | Purpose |
|----------|------|---------|
| `S_Update()` | `snd_dma.c` | Update listener position, spatialize all channels |
| `SND_Spatialize()` | `snd_dma.c` | Calculate left/right volume from 3D position |
| `S_PaintChannels()` | `snd_mix.c` | Mix all active channels into DMA buffer |

---

## Configuration

| CVar | Default | Purpose |
|------|---------|---------|
| `volume` | 0.7 | Master sound effects volume |
| `bgmvolume` | 1.0 | CD/background music volume |
| `nosound` | 0 | Disable all sound |
| `ambient_level` | 0.3 | Ambient sound volume |
| `ambient_fade` | 100 | Ambient sound fade rate |
