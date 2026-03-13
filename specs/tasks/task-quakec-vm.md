# Task: QuakeC Virtual Machine

> Technical implementation specification with file references.

---

## Subsystem Overview

The QuakeC VM is a bytecode interpreter that executes compiled game logic. It bridges between the engine (C) and game scripts (QuakeC) through ~100 builtin functions.

---

## File Inventory

### Engine VM Files (`legacy-src/desktop-engine/`)

| File | LOC (approx.) | Responsibility |
|------|---------------|----------------|
| `pr_exec.c` | 400 | Bytecode execution loop (`PR_ExecuteProgram()`) |
| `pr_edict.c` | 1,100 | Entity allocation, field access, progs loading |
| `pr_cmds.c` | 1,500 | ~100 builtin functions (engine bridge) |
| `pr_comp.h` | 180 | Opcode definitions, instruction format |
| `progs.h` | 130 | VM data structures |
| `progdefs.h` | — | Auto-generated entity field offsets |

### QuakeC Source Files (`legacy-src/qw-qc/`)

| File | LOC | Responsibility |
|------|-----|----------------|
| `defs.qc` | 754 | Global definitions, entity fields, builtin declarations |
| `subs.qc` | 310 | Movement utilities (`SUB_CalcMove`), target-firing (`SUB_UseTargets`) |
| `combat.qc` | 350 | Damage system (`T_Damage`, `T_RadiusDamage`, `Killed`) |
| `items.qc` | 1,686 | All pickups: weapons, ammo, armor, health, keys, powerups |
| `weapons.qc` | 1,427 | 7 weapon implementations + projectile callbacks |
| `world.qc` | 432 | World initialization, asset precaching, light styles |
| `client.qc` | 1,524 | Player spawning, death, respawn, level transitions |
| `spectate.qc` | 111 | Spectator mode |
| `player.qc` | 736 | Player animation frame definitions |
| `doors.qc` | 809 | Door state machine (open/close/blocked) |
| `buttons.qc` | 166 | Pushbutton mechanics |
| `triggers.qc` | 672 | 10 trigger types (teleport, hurt, push, relay, etc.) |
| `plats.qc` | 393 | Platforms (vertical) and trains (path-following) |
| `misc.qc` | 756 | Lights, fireballs, explosive barrels |
| `models.qc` | 611 | Model entity spawning helpers |
| `sprites.qc` | 52 | Sprite entity spawning |
| `server.qc` | 123 | Monster waypoint pathing (`path_corner`) |

### Compilation Control

| File | Purpose |
|------|---------|
| `legacy-src/qw-qc/progs.src` | Compilation order (output: `qwprogs.dat`) |
| `legacy-src/qw-qc/qwprogs.dat` | Compiled bytecode binary |
| `legacy-src/qw-qc/files.dat` | File listing |
| `legacy-src/qw-qc/progdefs.h` | Auto-generated C struct matching QuakeC fields |

---

## VM Execution Details

### Instruction Format
```c
typedef struct {
    unsigned short op;      // Opcode (0-79+)
    unsigned short a, b, c; // Global variable indices (operands)
} dstatement_t;
```

### Execution Stack
```c
#define MAX_STACK_DEPTH 32
typedef struct {
    int s;                  // Statement index (program counter)
    dfunction_t *f;         // Function being executed
} prstack_t;
```

### Global Variable Space
- All QuakeC variables stored in flat `float *pr_globals` array
- Entity fields accessed via offset into `entvars_t` structure
- Strings stored in `char *pr_strings` table (referenced by offset)
- Functions stored in `dfunction_t *pr_functions` array

### Execution Flow (`PR_ExecuteProgram()` — `legacy-src/desktop-engine/pr_exec.c`)
1. Enter function: push call stack, copy parameters to locals
2. Fetch statement at program counter
3. Switch on opcode, execute operation
4. Advance program counter
5. Loop until `OP_RETURN` or `OP_DONE`
6. Leave function: pop call stack, restore locals

---

## Builtin Function Categories

### Entity Management
| # | Function | C Implementation |
|---|----------|-----------------|
| 2 | `setorigin(e, o)` | Sets entity origin + re-links |
| 3 | `setmodel(e, m)` | Sets model, mins/maxs |
| 4 | `setsize(e, min, max)` | Sets collision bounds |
| 14 | `spawn()` | `ED_Alloc()` |
| 15 | `remove(e)` | `ED_Free()` |
| 18 | `find(start, fld, match)` | Entity search by field |
| 22 | `findradius(org, rad)` | Sphere query |

### Physics & Traces
| # | Function | Purpose |
|---|----------|---------|
| 16 | `traceline(v1, v2, nm, fe)` | Ray trace through world |
| 32 | `walkmove(yaw, dist)` | Move with collision |
| 34 | `droptofloor()` | Drop to ground |
| 40 | `checkbottom(e)` | Check ground contact |
| 67 | `movetogoal(step)` | Monster AI movement |

### Sound & Effects
| # | Function | Purpose |
|---|----------|---------|
| 8 | `sound(e, chan, samp, vol, atten)` | Play 3D sound |
| 19 | `precache_sound(s)` | Load sound file |
| 20 | `precache_model(s)` | Load model file |
| 35 | `lightstyle(style, value)` | Set light animation |

### Math
| # | Function | Purpose |
|---|----------|---------|
| 1 | `makevectors(angles)` | Angles → forward/right/up |
| 7 | `random()` | Random 0.0–1.0 |
| 9 | `normalize(v)` | Unit vector |
| 12 | `vlen(v)` | Vector length |
| 13 | `vectoyaw(v)` | Angle from vector |
| 49 | `ChangeYaw()` | Turn toward ideal_yaw |

### Messaging
| # | Function | Purpose |
|---|----------|---------|
| 23 | `bprint(level, s)` | Broadcast message |
| 24 | `sprint(client, level, s)` | Player message |
| 73 | `centerprint(client, s)` | Center screen text |
| 21 | `stuffcmd(client, s)` | Send command to client |
| 52-59 | `WriteXxx(to, f)` | Network message writing |
| 82 | `multicast(pos, set)` | Selective broadcast |

---

## Entity Callback System

The engine calls QuakeC through these entity callbacks:

| Field | When Called | Set By |
|-------|-----------|--------|
| `think` | `time >= self.nextthink` | `self.think = func; self.nextthink = time + delay;` |
| `touch` | Two solid entities collide | `self.touch = func;` |
| `use` | Entity targeted by `SUB_UseTargets()` | `self.use = func;` |
| `blocked` | `MOVETYPE_PUSH` entity hits obstacle | `self.blocked = func;` |
| `th_pain` | Entity takes damage (alive) | `self.th_pain = func;` |
| `th_die` | Entity dies (health ≤ 0) | `self.th_die = func;` |

---

## Compiler

- **Tool**: `qcc` (QuakeC compiler) — NOT included in this repository
- **Input**: `.qc` files in order specified by `progs.src`
- **Output**: `qwprogs.dat` (bytecode binary)
- **Compilation order**: `progs.src` lists files in dependency order
