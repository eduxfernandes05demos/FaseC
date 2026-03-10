# Feature: QuakeC Scripting Engine

> Reverse-engineered from `Quake/WinQuake/pr_exec.c`, `Quake/WinQuake/pr_edict.c`, `Quake/WinQuake/pr_cmds.c`, and `Quake/qw-qc/`

---

## Overview

QuakeC is a custom scripting language designed by John Carmack for Quake. It allows all game logic (weapons, items, enemies, triggers, level mechanics) to be defined in script rather than compiled into the engine. The engine contains a bytecode interpreter that executes compiled QuakeC programs (`progs.dat` / `qwprogs.dat`).

---

## Key Source Files

### Engine (VM Implementation)
| File | Purpose |
|------|---------|
| `Quake/WinQuake/pr_exec.c` | Bytecode execution loop — `PR_ExecuteProgram()` |
| `Quake/WinQuake/pr_edict.c` | Entity allocation, field access, progs loading — `ED_Alloc()`, `PR_LoadProgs()` |
| `Quake/WinQuake/pr_cmds.c` | ~100 builtin functions bridging script to engine |
| `Quake/WinQuake/pr_comp.h` | Opcode definitions (~80 opcodes), instruction format |
| `Quake/WinQuake/progs.h` | VM data structures (`edict_t`, `dstatement_t`, `dfunction_t`) |

### Game Logic (QuakeC Source)
| File | Lines | Purpose |
|------|-------|---------|
| `Quake/qw-qc/defs.qc` | 754 | Global definitions, entity fields, builtin declarations |
| `Quake/qw-qc/subs.qc` | 310 | Movement utilities, target-firing system |
| `Quake/qw-qc/combat.qc` | 350 | Damage system |
| `Quake/qw-qc/items.qc` | 1,686 | All pickups |
| `Quake/qw-qc/weapons.qc` | 1,427 | Weapon implementations |
| `Quake/qw-qc/world.qc` | 432 | World initialization, precaching |
| `Quake/qw-qc/client.qc` | 1,524 | Player management |
| `Quake/qw-qc/spectate.qc` | 111 | Spectator mode |
| `Quake/qw-qc/player.qc` | 736 | Player animation frames |
| `Quake/qw-qc/doors.qc` | 809 | Door mechanics |
| `Quake/qw-qc/buttons.qc` | 166 | Button mechanics |
| `Quake/qw-qc/triggers.qc` | 672 | Trigger entities |
| `Quake/qw-qc/plats.qc` | 393 | Platforms and trains |
| `Quake/qw-qc/misc.qc` | 756 | Lights, effects, barrels |
| `Quake/qw-qc/models.qc` | 611 | Model entity spawning |
| `Quake/qw-qc/sprites.qc` | 52 | Sprite entities |
| `Quake/qw-qc/server.qc` | 123 | Monster waypoint pathing |

---

## VM Architecture

### Instruction Format (`Quake/WinQuake/pr_comp.h`)
```c
typedef struct {
    unsigned short op;      // Opcode (0-79)
    unsigned short a, b, c; // Operand indices into global variables
} dstatement_t;
```

### Opcode Categories (~80 opcodes)

| Category | Opcodes | Description |
|----------|---------|-------------|
| Arithmetic | `OP_MUL_F`, `OP_MUL_V`, `OP_MUL_FV`, `OP_MUL_VF`, `OP_DIV_F`, `OP_ADD_F`, `OP_ADD_V`, `OP_SUB_F`, `OP_SUB_V` | Float and vector math |
| Comparison | `OP_EQ_F`, `OP_EQ_V`, `OP_EQ_S`, `OP_EQ_E`, `OP_EQ_FNC`, `OP_NE_*`, `OP_LE`, `OP_GE`, `OP_LT`, `OP_GT` | Type-safe comparisons |
| Logic | `OP_AND`, `OP_OR`, `OP_NOT_F`, `OP_NOT_V`, `OP_NOT_S`, `OP_NOT_E`, `OP_NOT_FNC` | Logical operations |
| Load/Store | `OP_LOAD_F`, `OP_LOAD_V`, `OP_LOAD_S`, `OP_LOAD_ENT`, `OP_LOAD_FLD`, `OP_LOAD_FNC`, `OP_STORE_*`, `OP_STOREP_*` | Variable access |
| Address | `OP_ADDRESS` | Get field pointer |
| Control | `OP_IF`, `OP_IFNOT`, `OP_GOTO` | Conditional/unconditional jumps |
| Functions | `OP_CALL0`–`OP_CALL8`, `OP_RETURN`, `OP_DONE` | Function calls with 0-8 parameters |
| Bitwise | `OP_BITAND`, `OP_BITOR` | Bitwise AND/OR |

### Execution Loop (`Quake/WinQuake/pr_exec.c`)

**`PR_ExecuteProgram(func_t fnum)`** (line ~361):
1. `PR_EnterFunction()` — Push current state onto call stack, set up locals
2. Fetch instruction at current statement pointer
3. Decode opcode and operand indices
4. Execute operation (switch on opcode)
5. Advance statement pointer
6. Repeat until `OP_RETURN` or `OP_DONE`
7. `PR_LeaveFunction()` — Pop call stack, restore previous function

**Stack**: `MAX_STACK_DEPTH = 32` — maximum call depth

### Data Types

| QuakeC Type | C Representation | Size |
|-------------|-----------------|------|
| `float` | `float` | 4 bytes |
| `vector` | `float[3]` | 12 bytes |
| `string` | `int` (offset into string table) | 4 bytes |
| `entity` | `int` (edict index) | 4 bytes |
| `void` | — | 0 bytes |
| function | `int` (function index) | 4 bytes |

### Entity Structure (`Quake/WinQuake/progs.h`)
```c
typedef struct edict_s {
    qboolean    free;           // Available for reuse?
    link_t      area;           // Linked into world area grid
    entity_state_t baseline;    // Network baseline state
    float       freetime;       // When entity was freed
    entvars_t   v;              // QuakeC-accessible fields
} edict_t;
```

---

## Builtin Functions (`Quake/WinQuake/pr_cmds.c`)

Key bridge functions between QuakeC and engine:

| # | QuakeC Name | Engine Function | Purpose |
|---|-------------|-----------------|---------|
| 1 | `makevectors(angles)` | `AngleVectors()` | Convert angles to direction vectors |
| 2 | `setorigin(e, o)` | Sets entity position | Position entity in world |
| 3 | `setmodel(e, m)` | Sets model and size | Assign visual model |
| 4 | `setsize(e, min, max)` | Sets bounding box | Define collision bounds |
| 7 | `random()` | — | Random float 0.0–1.0 |
| 8 | `sound(e, chan, samp, vol, atten)` | `SV_StartSound()` | Play 3D positioned sound |
| 9 | `normalize(v)` | — | Unit vector |
| 12 | `vlen(v)` | — | Vector magnitude |
| 14 | `spawn()` | `ED_Alloc()` | Create new entity |
| 15 | `remove(e)` | `ED_Free()` | Delete entity |
| 16 | `traceline(v1, v2, nm, fe)` | `SV_Move()` | Ray trace through world |
| 18 | `find(start, fld, match)` | — | Find entity by field value |
| 19 | `precache_sound(s)` | — | Load sound file |
| 20 | `precache_model(s)` | — | Load model file |
| 22 | `findradius(org, rad)` | — | Find entities in sphere |
| 23 | `bprint(level, s)` | — | Broadcast message |
| 24 | `sprint(client, level, s)` | — | Message to specific player |
| 32 | `walkmove(yaw, dist)` | `SV_movestep()` | Move with collision (AI) |
| 34 | `droptofloor()` | — | Drop entity to ground |
| 44 | `aim(e, speed)` | — | Auto-aim prediction |
| 45 | `cvar(s)` | `Cvar_VariableValue()` | Read console variable |
| 49 | `ChangeYaw()` | — | Turn toward ideal_yaw |
| 67 | `movetogoal(step)` | `SV_MoveToGoal()` | Monster movement toward goal |
| 73 | `centerprint(client, s)` | — | Center screen text |
| 80 | `infokey(e, key)` | — | Get entity info string |
| 82 | `multicast(pos, set)` | — | Broadcast to subset of clients |

---

## Entity Callback System

The engine calls QuakeC functions through the entity callback system:

| Callback | When Called | Example |
|----------|------------|---------|
| `think` | When `time >= self.nextthink` | `door_go_up()`, `player_run()` |
| `touch` | When two solid entities collide | `armor_touch()`, `spike_touch()` |
| `use` | When entity is targeted/triggered | `door_use()`, `button_use()` |
| `blocked` | When MOVETYPE_PUSH entity hits obstacle | `door_blocked()`, `plat_crush()` |
| `th_pain` | When entity takes damage (alive) | Pain animations |
| `th_die` | When entity dies (health ≤ 0) | Death animations, gibs |

---

## Compilation

**Input**: `.qc` files listed in `Quake/qw-qc/progs.src`
**Output**: `qwprogs.dat` — binary bytecode
**Compiler**: `qcc` (QuakeC compiler, not included in this repository)

Compilation order defined in `progs.src`:
1. `defs.qc` → 2. `subs.qc` → 3. `combat.qc` → ... → 18. `server.qc`

---

## Limitations

- No arrays or dynamic data structures
- No string manipulation (concatenation, substring, etc.)
- No integer type (all numbers are floats)
- Maximum 32 nested function calls
- No debugging facilities (no breakpoints, limited error messages)
- Performance overhead vs. native C code (interpreted bytecode)
- Single-threaded execution only
- No module system (all files compiled into one program)
- QuakeC compiler (`qcc`) not included in this repository
