# Task: Game Logic Implementation

> Technical implementation specification with file references.

---

## Subsystem Overview

All gameplay mechanics are implemented in QuakeC scripts compiled into `qwprogs.dat`. This task covers the implementation details of each gameplay system.

---

## File Inventory (`legacy-src/qw-qc/`)

| File | LOC | Primary Systems |
|------|-----|-----------------|
| `defs.qc` | 754 | Global constants, entity field definitions, builtin function declarations |
| `subs.qc` | 310 | `SUB_CalcMove()`, `SUB_CalcAngleMove()`, `SUB_UseTargets()`, `SetMovedir()` |
| `combat.qc` | 350 | `T_Damage()`, `T_RadiusDamage()`, `T_BeamDamage()`, `Killed()`, `CanDamage()` |
| `items.qc` | 1,686 | Health, armor, ammo, weapon, key, powerup, sigil pickups |
| `weapons.qc` | 1,427 | 7 weapon fire functions, projectile entities, ammo management |
| `world.qc` | 432 | `main()`, `worldspawn()`, `StartFrame()`, body queue, precaching |
| `client.qc` | 1,524 | `PutClientInServer()`, `respawn()`, `PlayerDie()`, `ClientObituary()`, level transitions |
| `spectate.qc` | 111 | `SpectatorConnect/Disconnect/Think/ImpulseCommand()` |
| `player.qc` | 736 | Player animation frame definitions (`$frame` directives) |
| `doors.qc` | 809 | `func_door()`, `func_door_secret()`, door state machine, `LinkDoors()` |
| `buttons.qc` | 166 | `func_button()`, button press/return cycle |
| `triggers.qc` | 672 | 10 trigger types: multiple, once, relay, secret, counter, teleport, hurt, push, monsterjump, setskill |
| `plats.qc` | 393 | `func_plat()`, `func_train()`, path_corner following |
| `misc.qc` | 756 | Light entities (10+ types), `misc_fireball()`, `misc_explobox()` |
| `models.qc` | 611 | Model entity spawning helpers |
| `sprites.qc` | 52 | Sprite entity spawning |
| `server.qc` | 123 | `path_corner`, `t_movetarget()` for monster waypoints |

---

## Weapons Implementation (`weapons.qc`)

### Weapon Registry

| Weapon | Flag | Function | Damage | Ammo |
|--------|------|----------|--------|------|
| Axe | `IT_AXE` | `W_FireAxe()` | 20 (melee trace 64 units) | None |
| Shotgun | `IT_SHOTGUN` | `W_FireShotgun()` | 6 pellets × 4 = 24 | 1 shell |
| Super Shotgun | `IT_SUPER_SHOTGUN` | `W_FireSuperShotgun()` | 14 pellets × 4 = 56 | 2 shells |
| Nailgun | `IT_NAILGUN` | `W_FireSpikes()` | 9 per nail | 1 nail |
| Super Nailgun | `IT_SUPER_NAILGUN` | `W_FireSuperSpikes()` | 18 per nail | 2 nails |
| Grenade Launcher | `IT_GRENADE_LAUNCHER` | `W_FireGrenade()` | 100-120 (radius) | 1 rocket |
| Rocket Launcher | `IT_ROCKET_LAUNCHER` | `W_FireRocket()` | 100-120 (radius) | 1 rocket |
| Lightning Gun | `IT_LIGHTNING` | `W_FireLightning()` | 30/frame (beam) | 1 cell/frame |

### Projectile Entity Setup Pattern
```
entity missile = spawn();
missile.owner = self;               // Prevent self-collision
missile.movetype = MOVETYPE_FLYMISSILE;  // Or MOVETYPE_BOUNCE for grenades
missile.solid = SOLID_BBOX;
missile.classname = "rocket";       // Or "grenade", "spike"
missile.touch = T_MissileTouch;     // Impact callback
setmodel(missile, "progs/missile.mdl");
setsize(missile, '0 0 0', '0 0 0');
setorigin(missile, self.origin + '0 0 16');
missile.velocity = aim(self, 1000) * 1000;
missile.nextthink = time + 5;       // Self-destruct timer
missile.think = SUB_Remove;
```

---

## Damage System (`combat.qc`)

### `T_Damage()` Flow
```
1. if (targ.takedamage == DAMAGE_NO) return;
2. armor_save = ceil(targ.armortype * damage)
3. if (armor_save >= targ.armorvalue) armor_save = targ.armorvalue
4. targ.armorvalue -= armor_save
5. remaining = damage - armor_save
6. if (attacker has IT_QUAD) remaining *= 4  // or 8 on dm4
7. if (targ.flags & FL_GODMODE) return
8. targ.health -= remaining
9. targ.velocity += direction * damage * 8  // Knockback
10. if (targ.health <= 0) Killed(targ, attacker)
11. else targ.th_pain(attacker, remaining)
```

### `T_RadiusDamage()` Flow
```
1. head = findradius(origin, damage + 40)
2. while (head):
3.   if (CanDamage(head, inflictor)):
4.     dist = vlen(head.origin - origin)
5.     points = damage - dist * 0.5  // Distance falloff
6.     if (head == attacker) points *= 0.5  // Self-damage reduction
7.     T_Damage(head, inflictor, attacker, points)
8.   head = head.chain
```

---

## Item Pickup Pattern (`items.qc`)

```
// Spawn function (called by map loader)
void() item_health = {
    self.touch = health_touch;
    precache_model("maps/b_bh25.bsp");
    setmodel(self, "maps/b_bh25.bsp");
    setsize(self, '0 0 0', '32 32 56');
    StartItem();
};

// Touch callback (called on collision)
void() health_touch = {
    if (other.classname != "player") return;
    if (other.health >= other.max_health) return;
    other.health = min(other.health + 25, other.max_health);
    sprint(other, PRINT_LOW, "You receive 25 health\n");
    sound(other, CHAN_ITEM, "items/health1.wav", 1, ATTN_NORM);
    // In deathmatch, schedule respawn
    self.model = string_null;    // Hide item
    self.solid = SOLID_NOT;      // No collision
    self.nextthink = time + 20;  // Respawn in 20s
    self.think = SUB_regen;
};
```

---

## Door State Machine (`doors.qc`)

```
STATE_BOTTOM → (touch/use) → STATE_UP → STATE_TOP → (wait) → STATE_DOWN → STATE_BOTTOM
                                 ↑                                  ↑
                            door_go_up()                     door_go_down()
                            SUB_CalcMove(pos2)              SUB_CalcMove(pos1)
```

### Linked Doors
- `LinkDoors()` finds adjacent doors (touching bounding boxes)
- Links them via `enemy` chain
- All linked doors fire together when any one is activated

---

## Trigger Architecture (`triggers.qc`)

### Activation Chain
```
trigger_multiple
  → multi_touch() [player enters trigger volume]
    → multi_trigger()
      → SUB_UseTargets() [fire all entities with matching targetname]
        → target.use() [call use function on each target]
      → multi_wait() [set cooldown before re-triggering]
```

### Teleport Mechanics
```
trigger_teleport.touch = teleport_touch
  1. Find info_teleport_destination by target name
  2. Play teleport sound + visual effect at both ends
  3. setorigin(other, destination.origin)
  4. other.angles = destination.mangle
  5. other.fixangle = TRUE  // Force view angle
  6. other.teleport_time = time + 0.7  // Prevent telefrag
  7. other.velocity = v_forward * 300  // Exit velocity
```

---

## Player Lifecycle (`client.qc`)

```
ClientConnect() → SetNewParms() → PutClientInServer()
  [playing: PlayerPreThink → physics → PlayerPostThink per frame]
  PlayerDie() → DeathThink() → respawn() → PutClientInServer()
  trigger_changelevel → SetChangeParms() → [next map] → DecodeLevelParms()
ClientDisconnect()
```

### Default Spawn Equipment
- Weapons: `IT_SHOTGUN | IT_AXE`
- Health: 100
- Armor: 0
- Ammo: 25 shells

---

## World Initialization (`world.qc`)

### `worldspawn()` precaches:
- **Player models**: `progs/player.mdl`, `progs/eyes.mdl`, 3 gib models
- **Weapon view models**: `progs/v_axe.mdl`, `v_shot.mdl`, `v_nail.mdl`, etc.
- **Projectiles**: `progs/missile.mdl`, `progs/grenade.mdl`, `progs/spike.mdl`, `progs/bolt.mdl`
- **Sounds**: ~50 sounds (weapons, player, items, ambient, UI)
- **Light styles**: 12 predefined animation patterns

### `StartFrame()` per frame:
- Updates `timelimit`, `fraglimit`, `teamplay`, `deathmatch` from cvars
- Increments `framecount`
