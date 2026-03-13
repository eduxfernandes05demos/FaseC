# Task: Input Subsystem

> Technical implementation specification with file references.

---

## Subsystem Overview

The input subsystem provides keyboard, mouse, and joystick input across all supported platforms through a common interface.

---

## File Inventory (`legacy-src/desktop-engine/`)

### Platform Drivers
| File | Platform | Devices |
|------|----------|---------|
| `in_win.c` | Windows | Mouse (DirectInput), joystick (6-axis), keyboard |
| `in_dos.c` | DOS | Mouse, keyboard via interrupts |
| `in_sun.c` | Solaris | X11 mouse/keyboard events |
| `in_null.c` | Any | Null stub driver |

### Key System
| File | Responsibility |
|------|----------------|
| `keys.c` | Key binding, key event dispatch, console input, history |
| `keys.h` | Key code definitions (K_TAB, K_ENTER, K_ESCAPE, K_MOUSE1, etc.) |

### Interface
| File | Responsibility |
|------|----------------|
| `input.h` | Public interface: `IN_Init()`, `IN_Move()`, `IN_Commands()`, `IN_Shutdown()` |

---

## Interface Functions (`legacy-src/desktop-engine/input.h`)

| Function | Called By | Purpose |
|----------|----------|---------|
| `IN_Init()` | Engine startup | Initialize input devices |
| `IN_Shutdown()` | Engine shutdown | Release input devices |
| `IN_Commands()` | Each frame | Generate button events from device state |
| `IN_Move(usercmd_t *cmd)` | Each frame | Read mouse/joystick into movement command |
| `IN_ClearStates()` | Mode change | Reset all input state |

---

## Key Event Flow

```
Platform input event (e.g., WM_KEYDOWN on Windows)
  → Key_Event(int key, qboolean down) [keys.c]
    → Check current key destination:
       ├── KEY_CONSOLE: Feed to console input line
       ├── KEY_MENU: Feed to menu system
       └── KEY_GAME: Execute keybinding
           → Cbuf_AddText(keybindings[key])
```

### Key Destinations
| Destination | Active When | Behavior |
|-------------|------------|----------|
| `KEY_GAME` | Playing | Execute bound commands |
| `KEY_CONSOLE` | Console open | Type into console |
| `KEY_MESSAGE` | Chat open | Type chat message |
| `KEY_MENU` | Menu open | Menu navigation |

---

## Mouse Input (Windows — `in_win.c`)

### Modes
- **Windowed**: Mouse free, position tracked
- **Fullscreen**: Mouse captured, relative motion via DirectInput or GetCursorPos delta
- **Menu**: Mouse cursor visible for UI interaction

### DirectInput
- Exclusive access in fullscreen mode
- Relative motion data (no cursor bounds issues)
- Buffered device data via `IDirectInputDevice::GetDeviceData()`

### Mouse CVars
| CVar | Purpose |
|------|---------|
| `sensitivity` | Mouse sensitivity multiplier |
| `m_pitch` | Mouse pitch (vertical) scale |
| `m_yaw` | Mouse yaw (horizontal) scale |
| `m_forward` | Mouse forward movement scale |
| `m_side` | Mouse side movement scale |
| `m_filter` | Mouse smoothing (average 2 frames) |

---

## Joystick Input (Windows — `in_win.c`)

- 6 axes: X, Y, Z, R (rotation), U, V
- Configurable axis mapping to: Forward, Look, Side, Turn
- CVars for sensitivity, dead zones, and advanced mapping
- Windows multimedia joystick API (`joyGetPosEx`)

---

## User Command Output (`legacy-src/desktop-engine/protocol.h`)

```c
typedef struct {
    byte    msec;           // Milliseconds since last command
    byte    buttons;        // Button bitmask
    short   forwardmove;    // Forward/back (-127 to 127)
    short   sidemove;       // Left/right strafe
    short   upmove;         // Up/down (swimming/flying)
    vec3_t  angles;         // View angles
} usercmd_t;
```
