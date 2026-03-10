# Feature: Game Logic — Weapons, Items, AI, Triggers

> Reverse-engineered from `Quake/qw-qc/*.qc`

---

## Overview

All gameplay mechanics in Quake are implemented in QuakeC scripts, compiled into `qwprogs.dat`. This includes weapons, items, combat, player mechanics, map entities (doors, buttons, triggers, platforms), and world initialization. The QuakeWorld version does **not** include monster AI (only player-vs-player gameplay).

---

## Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `Quake/qw-qc/weapons.qc` | 1,427 | All weapon implementations |
| `Quake/qw-qc/items.qc` | 1,686 | Health, armor, ammo, keys, powerups |
| `Quake/qw-qc/combat.qc` | 350 | Damage system (`T_Damage`, `T_RadiusDamage`) |
| `Quake/qw-qc/client.qc` | 1,524 | Player spawning, death, level transitions |
| `Quake/qw-qc/doors.qc` | 809 | Door mechanics (linked, keyed, secret) |
| `Quake/qw-qc/triggers.qc` | 672 | Trigger entities (teleport, hurt, push, relay) |
| `Quake/qw-qc/plats.qc` | 393 | Platforms and trains |
| `Quake/qw-qc/buttons.qc` | 166 | Pushbutton mechanics |
| `Quake/qw-qc/subs.qc` | 310 | Movement utilities, target-firing system |
| `Quake/qw-qc/world.qc` | 432 | World initialization, asset precaching |
| `Quake/qw-qc/player.qc` | 736 | Player animation frame definitions |
| `Quake/qw-qc/spectate.qc` | 111 | Spectator mode |
| `Quake/qw-qc/misc.qc` | 756 | Lights, fireballs, explosive barrels |
| `Quake/qw-qc/defs.qc` | 754 | Global definitions, entity fields, builtins |
| `Quake/qw-qc/server.qc` | 123 | Monster waypoint pathing |
| `Quake/qw-qc/models.qc` | 611 | Model entity spawning |
| `Quake/qw-qc/sprites.qc` | 52 | Sprite entities |

---

## Weapons System (`Quake/qw-qc/weapons.qc`)

### Available Weapons

| Weapon | Item Flag | Ammo Type | Ammo/Shot |
|--------|-----------|-----------|-----------|
| Axe | `IT_AXE` (4096) | None | — |
| Shotgun | `IT_SHOTGUN` (1) | Shells | 1 |
| Super Shotgun | `IT_SUPER_SHOTGUN` (2) | Shells | 2 |
| Nailgun | `IT_NAILGUN` (4) | Nails | 1 |
| Super Nailgun | `IT_SUPER_NAILGUN` (8) | Nails | 2 |
| Grenade Launcher | `IT_GRENADE_LAUNCHER` (16) | Rockets | 1 |
| Rocket Launcher | `IT_ROCKET_LAUNCHER` (32) | Rockets | 1 |
| Lightning Gun | `IT_LIGHTNING` (64) | Cells | 1/frame |

### Key Weapon Functions

| Function | Description |
|----------|-------------|
| `W_Attack()` | Main attack dispatcher — checks ammo, selects weapon handler |
| `W_FireAxe()` | Melee: 64-unit trace, 20 damage |
| `W_FireShotgun()` | 6 pellets, spread pattern |
| `W_FireSuperShotgun()` | 14 pellets, wider spread |
| `W_FireSpikes()` | Nailgun projectile (9 damage each) |
| `W_FireSuperSpikes()` | Super nailgun projectile (18 damage each) |
| `W_FireRocket()` | Rocket entity: MOVETYPE_FLYMISSILE, 100-120 explosion damage |
| `W_FireGrenade()` | Grenade entity: MOVETYPE_BOUNCE, 2.5s fuse, 120 explosion damage |
| `W_FireLightning()` | Continuous beam: 30 damage/frame, `traceline` based |
| `W_SetCurrentAmmo()` | Update current ammo display based on weapon |
| `W_ChangeWeapon()` | Weapon switching logic |
| `W_Precache()` | Precache all weapon sounds |

### Projectile Callbacks

| Function | Trigger | Effect |
|----------|---------|--------|
| `T_MissileTouch()` | Rocket hits surface/entity | Explosion + `T_RadiusDamage(120)` |
| `spike_touch()` | Nail hits surface/entity | Direct damage + impact effect |
| `GrenadeTouch()` | Grenade hits surface | Bounce or explode based on target |
| `GrenadeExplode()` | Grenade timer (2.5s) | Explosion + `T_RadiusDamage(120)` |

---

## Combat System (`Quake/qw-qc/combat.qc`)

### `T_Damage(targ, inflictor, attacker, damage)`
1. Check `takedamage` flag
2. Apply armor reduction: `save = ceil(armortype * damage)`
3. Apply Quad Damage: 4× multiplier (8× on dm4 map)
4. Check team damage protection (`teamplay` cvar)
5. Check invulnerability/god mode
6. Reduce health by remaining damage
7. Apply knockback: `velocity += direction * damage * 8`
8. Call `th_pain()` if alive, `Killed()` if dead

### `T_RadiusDamage(inflictor, attacker, damage, ignore, dtype)`
1. Find all entities within radius using `findradius()`
2. For each: calculate distance-reduced damage
3. Check line-of-sight via `CanDamage()`
4. 50% self-damage reduction
5. Apply `T_Damage()` to each target

---

## Items System (`Quake/qw-qc/items.qc`)

### Item Categories

| Category | Items |
|----------|-------|
| **Health** | Small (+25), Mega (+100, regenerating) |
| **Armor** | Green (0.3 protection), Yellow (0.6), Red (0.8) |
| **Ammo** | Shells (20/40), Nails (25/50), Rockets (5/10), Cells (6/12) |
| **Weapons** | Super Shotgun, Nailgun, Super Nailgun, Grenade Launcher, Rocket Launcher, Lightning Gun |
| **Keys** | Gold key, Silver key |
| **Powerups** | Quad Damage (30s), Invulnerability, Invisibility, Biosuit |
| **Sigils** | 4 end-of-episode runes |

### Item Mechanics
- Items use `SOLID_TRIGGER` — picked up on touch
- `SUB_regen()` respawns items after a delay (deathmatch mode)
- `PlaceItem()` / `StartItem()` configure item entity defaults
- Items set `self.touch` to pickup handler function

---

## Map Entities

### Doors (`Quake/qw-qc/doors.qc`)

**States**: `STATE_TOP (0)`, `STATE_BOTTOM (1)`, `STATE_UP (2)`, `STATE_DOWN (3)`

**Spawnflags**:
- `DOOR_START_OPEN (1)` — Starts in open position
- `DOOR_DONT_LINK (4)` — Don't link with adjacent doors
- `DOOR_GOLD_KEY (8)` — Requires gold key
- `DOOR_SILVER_KEY (16)` — Requires silver key
- `DOOR_TOGGLE (32)` — Toggle mode, don't auto-close

**Key functions**: `func_door()`, `door_use()`, `door_touch()`, `door_blocked()`, `LinkDoors()`

### Buttons (`Quake/qw-qc/buttons.qc`)
- Move from `pos1` (idle) to `pos2` (pressed)
- Fire targets via `SUB_UseTargets()` when pressed
- Auto-return after `wait` seconds
- **Key functions**: `func_button()`, `button_use()`, `button_fire()`

### Triggers (`Quake/qw-qc/triggers.qc`)

| Trigger | Purpose |
|---------|---------|
| `trigger_multiple` | Repeatable activation with cooldown |
| `trigger_once` | Single activation, then removed |
| `trigger_relay` | Passes activation to target without touching |
| `trigger_secret` | Counts as discovered secret |
| `trigger_counter` | Counts N activations before firing |
| `trigger_teleport` | Teleports touching entity to destination |
| `trigger_hurt` | Damage over time (lava/slime) |
| `trigger_push` | Applies velocity to touching entity |
| `trigger_monsterjump` | Launches monsters |
| `trigger_setskill` | Sets difficulty on touch |

### Platforms & Trains (`Quake/qw-qc/plats.qc`)
- **Platforms**: Move between `pos1` (bottom) and `pos2` (top)
- **Trains**: Follow `path_corner` waypoints linked by `target`
- Both use `MOVETYPE_PUSH` + `SOLID_BSP`
- **Key functions**: `func_plat()`, `func_train()`, `train_next()`

### Lights & Misc (`Quake/qw-qc/misc.qc`)
- Static lights: `light()`, `light_fluoro()`, `light_torch_small_walltorch()`
- Flames: `light_flame_large_yellow()`, `light_flame_small_yellow()`
- Explosive barrels: `misc_explobox()` (50 HP, chain explosions)
- Fireballs: `misc_fireball()` (flying fire projectile)

---

## Player System (`Quake/qw-qc/client.qc`)

### Player Lifecycle

```
ClientConnect()      — Player joins server
  ↓
PutClientInServer()  — Spawn into level
  ↓
[gameplay loop]      — PlayerPreThink / PlayerPostThink each frame
  ↓
PlayerDie()          — Death handler
  ↓
respawn()            — Respawn after death
  ↓
ClientDisconnect()   — Player leaves
```

### Starting Equipment
- Weapons: Axe + Shotgun
- Health: 100
- Ammo: 25 shells

### Level Transitions
- `trigger_changelevel` entity triggers map change
- `SetChangeParms()` saves player state to spawn parameters
- `DecodeLevelParms()` restores state in new level

---

## Spectator System (`Quake/qw-qc/spectate.qc`)

| Function | Purpose |
|----------|---------|
| `SpectatorConnect()` | Broadcasts "entered" message |
| `SpectatorDisconnect()` | Broadcasts "left" message |
| `SpectatorThink()` | Per-frame update (minimal) |
| `SpectatorImpulseCommand()` | Impulse 1: teleport to next spawn point |

Spectators: no collision, no weapons, no scoring. Can cycle through deathmatch spawn points.

---

## Target/Trigger System (`Quake/qw-qc/subs.qc`)

Central activation mechanism:

```
Entity A (target = "door1")
  → SUB_UseTargets()
    → find all entities with targetname = "door1"
    → call their .use() function
    → supports: delay, killtarget, message
```

### Movement Utilities
- `SUB_CalcMove(dest, speed, callback)` — Linear movement to destination
- `SUB_CalcAngleMove(angles, speed, callback)` — Rotation to angle
- `SetMovedir()` — Convert editor angles to movement vector

---

## Limitations

- **No monster AI**: QuakeWorld QuakeC does not include monster code (PvP only)
- **No cooperative AI**: No friendly NPCs or companion mechanics
- **No weapon balance tuning**: Hardcoded damage values, no config-driven balance
- **No dynamic item spawning**: Items placed at map compile time only
- **Limited QuakeC**: No arrays, no string manipulation, no debugging
- **Fixed 4-player-type spawn system**: info_player_start, deathmatch, coop, spectator
