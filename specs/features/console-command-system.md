# Feature: Console and Command System

> Reverse-engineered from `Quake/WinQuake/cmd.c`, `Quake/WinQuake/cvar.c`, `Quake/WinQuake/console.c`, `Quake/WinQuake/keys.c`

---

## Overview

The console and command system provides a unified text-based interface for configuring the engine, binding keys, executing scripts, and debugging. It processes commands from multiple sources (keyboard, config files, network, key bindings) through a single command buffer.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/WinQuake/cmd.c` | Command buffer, tokenization, command registration and execution |
| `Quake/WinQuake/cmd.h` | Command system interface |
| `Quake/WinQuake/cvar.c` | Console variable registration, get/set, archival |
| `Quake/WinQuake/cvar.h` | CVar structure definition |
| `Quake/WinQuake/console.c` | Console rendering, scrollback buffer, notification lines |
| `Quake/WinQuake/console.h` | Console interface |
| `Quake/WinQuake/keys.c` | Key binding, key event dispatch |
| `Quake/WinQuake/keys.h` | Key code definitions |

---

## Command Execution Pipeline

```
Input Sources:
  ├── Console typing     → Cbuf_AddText("command\n")
  ├── Config file        → Cbuf_InsertText(file_contents)
  ├── Key binding        → Cbuf_AddText(keybindings[key])
  ├── Network (stufftext)→ Cbuf_AddText(server_command)
  └── Aliases            → Cbuf_InsertText(alias_text)
       ↓
Cbuf_Execute() — called each frame
       ↓
Extract one line at a time
       ↓
Cmd_TokenizeString(line) — split into argc/argv
       ↓
Cmd_ExecuteString(line)
  ├── Check registered commands (Cmd_AddCommand) → call handler
  ├── Check console variables (Cvar_Command) → set value
  └── Check aliases → expand and re-execute
```

---

## Command Buffer (`Quake/WinQuake/cmd.c`)

| Function | Purpose |
|----------|---------|
| `Cbuf_Init()` | Initialize the command buffer |
| `Cbuf_AddText(char *text)` | Append text to end of buffer |
| `Cbuf_InsertText(char *text)` | Insert text at front of buffer (for immediate execution) |
| `Cbuf_Execute()` | Process all pending commands line-by-line |

The buffer is a simple text string. `Cbuf_Execute()` extracts lines (delimited by `\n` or `;`), tokenizes each, and dispatches.

---

## Command Registration

| Function | Purpose |
|----------|---------|
| `Cmd_AddCommand(char *name, xcommand_t func)` | Register a command with handler function |
| `Cmd_Argc()` | Get argument count for current command |
| `Cmd_Argv(int n)` | Get argument by index |
| `Cmd_Args()` | Get all arguments as single string |
| `Cmd_CompleteCommand(char *partial)` | Tab-completion |
| `Cmd_ExecuteString(char *text, cmd_source_t src)` | Execute a single command |

**Command sources** (`cmd_source_t`):
- `src_client` — From connected client
- `src_command` — From console or script

---

## Console Variables (CVars) (`Quake/WinQuake/cvar.c`)

### CVar Structure
```c
typedef struct cvar_s {
    char      *name;     // Variable name (e.g., "volume")
    char      *string;   // String value (e.g., "0.7")
    qboolean   archive;  // Save to config.cfg on exit
    qboolean   server;   // Notify clients on change
    float      value;    // Numeric value (auto-parsed from string)
    struct cvar_s *next; // Linked list
} cvar_t;
```

### CVar Functions

| Function | Purpose |
|----------|---------|
| `Cvar_RegisterVariable(cvar_t *var)` | Register new console variable |
| `Cvar_Set(char *name, char *value)` | Set variable by name |
| `Cvar_SetValue(char *name, float value)` | Set variable to numeric value |
| `Cvar_VariableValue(char *name)` | Get variable as float |
| `Cvar_VariableString(char *name)` | Get variable as string |
| `Cvar_CompleteVariable(char *partial)` | Tab-completion for variables |
| `Cvar_WriteVariables(int f)` | Write all archived cvars to file |

### Notable CVars

| CVar | Default | Purpose |
|------|---------|---------|
| `sv_gravity` | 800 | World gravity |
| `sv_maxspeed` | 320 | Maximum player speed |
| `sv_friction` | 4 | Ground friction |
| `sensitivity` | 3 | Mouse sensitivity |
| `volume` | 0.7 | Sound volume |
| `bgmvolume` | 1.0 | Music volume |
| `cl_forwardspeed` | 200 | Forward movement speed |
| `cl_backspeed` | 200 | Backward movement speed |
| `cl_sidespeed` | 350 | Strafe speed |
| `crosshair` | 0 | Show crosshair |
| `r_drawflat` | 0 | Flat-shaded rendering (debug) |
| `r_speeds` | 0 | Show rendering statistics |
| `host_speeds` | 0 | Show timing statistics |

---

## Key Binding System (`Quake/WinQuake/keys.c`)

### Key States
```c
char *keybindings[2][K_LAST];  // Bindings for each key (down/up)
qboolean keydown[K_LAST];      // Current key states
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `Key_Event(int key, qboolean down)` | Process key press/release event |
| `Key_SetBinding(int keynum, char *binding)` | Bind command to key |
| `Key_Unbind_f()` | Remove key binding |
| `Key_ClearStates()` | Reset all key states |

### Key Event Flow
```
Platform input (e.g., in_win.c)
  → Key_Event(K_MOUSE1, true)
    → Check key destination (console, menu, game)
    → If game: Cbuf_AddText(keybindings[K_MOUSE1])
      → e.g., "+attack" is added to command buffer
```

### Special Key Handling
- `+command` / `-command` pattern: `+attack` on key down, `-attack` on key up
- Console key (`` ` `` or `~`): Toggle console visibility
- Escape: Open/close menu
- Tab: Tab-completion in console

---

## Console Display (`Quake/WinQuake/console.c`)

### Features
- Drop-down overlay that slides from top of screen
- Scrollback buffer for viewing previous output
- Notification lines (temporary messages at top of screen)
- Text input with cursor and editing
- Tab completion for commands and variables
- History navigation (up/down arrows)

### Functions

| Function | Purpose |
|----------|---------|
| `Con_Init()` | Initialize console |
| `Con_Print(char *txt)` | Add text to console buffer |
| `Con_Printf(char *fmt, ...)` | Printf-style output to console |
| `Con_DPrintf(char *fmt, ...)` | Debug printf (only when `developer` cvar set) |
| `Con_DrawConsole(int lines)` | Render console overlay |
| `Con_DrawNotify()` | Render notification lines |
| `Con_CheckResize()` | Handle console width changes |

---

## Aliases

Aliases allow defining custom commands that expand to other commands:

```
alias rj "rocket_jump; +jump; wait; -jump"
```

When `rj` is typed, it expands to `rocket_jump; +jump; wait; -jump` and is re-executed through the command buffer.

---

## Config Files

- `config.cfg` — Saved automatically on exit, contains all archived cvars and key bindings
- `autoexec.cfg` — Executed on startup after `config.cfg`
- `exec filename.cfg` — Manually execute a config file

---

## Limitations

- No command-line history persistence (lost on exit)
- Tab completion shows only first match (no cycling through multiple matches)
- No syntax highlighting or colored output
- Fixed-width font only
- No mouse interaction with console
- No undo/redo for text input
