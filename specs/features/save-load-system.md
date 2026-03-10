# Feature: Save/Load System

> Reverse-engineered from `Quake/WinQuake/host_cmd.c`

---

## Overview

The save/load system allows players to persist and restore complete game state during single-player gameplay. It serializes all entity data, player state, and level information to disk files.

---

## Key Source Files

| File | Purpose |
|------|---------|
| `Quake/WinQuake/host_cmd.c` | Save and load implementation (`Host_Savegame_f`, `Host_Loadgame_f`) |
| `Quake/WinQuake/pr_edict.c` | Entity serialization (`ED_Write`, `ED_ParseEdict`) |
| `Quake/WinQuake/sv_main.c` | Server state management |
| `Quake/WinQuake/host.c` | Level transition support |

---

## Save Format

Save files are stored as human-readable text files in the game directory (e.g., `id1/s0.sav`).

### File Structure
```
<version number>        — Save format version
<description string>    — User-visible save name
<spawn parameters>      — Player carry-over state (16 floats)
<current skill>         — Difficulty level
<map name>              — Current level
<server time>           — Elapsed time
<lightstyle strings>    — Current light animation states
{                       — Entity block (repeated for each entity)
  "classname" "player"
  "origin" "128 256 64"
  "health" "100"
  ...
}
```

---

## Key Functions

### Saving (`Host_Savegame_f` in `Quake/WinQuake/host_cmd.c`)
1. Validate: must be in single-player, not a demo, client must be alive
2. Create save file in game directory
3. Write version, description, spawn parms, skill, map name, time
4. Write current light styles
5. For each entity: `ED_Write()` serializes all non-default fields
6. Close file

### Loading (`Host_Loadgame_f` in `Quake/WinQuake/host_cmd.c`)
1. Open save file
2. Read and validate version number
3. Read spawn parameters, skill, map name, time
4. Force disconnect current game
5. Load the saved map via `SV_SpawnServer()`
6. Restore server time
7. Restore light styles
8. Parse entity blocks: `ED_ParseEdict()` for each entity
9. Re-link entities into world
10. Restore client connection

---

## Console Commands

| Command | Purpose |
|---------|---------|
| `save <name>` | Save game to `<name>.sav` |
| `load <name>` | Load game from `<name>.sav` |

Typically bound to quick-save/quick-load keys:
```
bind F5 "save quick"
bind F9 "load quick"
```

---

## Limitations

- Single-player only (not available in multiplayer)
- Cannot save during multiplayer or demo playback
- Player must be alive to save
- No auto-save functionality
- Save files are human-readable text (no compression)
- No versioning or migration between engine versions
- Monster AI state is partially lost on load (think timers reset)
