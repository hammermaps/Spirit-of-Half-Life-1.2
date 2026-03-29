# Spirit of Half-Life 1.2a

**Spirit of Half-Life** (SoHL) is an extensive modification of the Half-Life GoldSrc engine SDK that adds a large number of new entities, behaviours, and mapping capabilities without replacing the original Half-Life gameplay. Version 1.2 was the last official release by its original author, **Laurie Cheers** (LRC).

This repository contains **Admer's port of SoHL 1.2 to Visual Studio 2019**, built on top of [Solokiller's updated HL SDK](https://github.com/Solokiller/halflife-updated).

---

## Features

All changes by Laurie Cheers are marked with `//LRC` comments throughout the source code. The sections below summarise every major feature area.

### MoveWith System
Entities can be attached to a parent entity and will move/rotate with it. Any entity can specify a `movewith` key pointing to another entity's name. Implemented in `movewith.cpp` / `movewith.h` and integrated into `CBaseEntity`.

- New fields: `m_MoveWith`, `m_pMoveWith`, `m_pChildMoveWith`, `m_pSiblingMoveWith`
- Offset tracking via `m_vecMoveWithOffset` and `m_vecRotWithOffset`
- Assist-list rebuild on game restore
- `UTIL_SetVelocity`, `UTIL_SetAvelocity`, `UTIL_SetAngles` utilities that propagate to children

### Locus System
A general-purpose position-reference system that allows entity parameters to reference positions in the world dynamically (the `$locus` keyword). Implemented in `locus.cpp` / `locus.h`.

- `calc_position` – computes a world position
- `calc_velocity_path` / `calc_velocity_polar` – compute velocity vectors
- `calc_ratio` – computes a numeric ratio
- `calc_subvelocity` – sub-velocity calculation
- `UTIL_StringToVector` extended with locus-aware parsing

### Alias System
Multi-purpose entity-aliasing system (`alias.cpp`).

- `multi_alias` – maps a name to different real entity names depending on state
- `trigger_changealias` – changes what an alias points to at runtime
- Alias list rebuilt on game restore

### State System
Every entity can now have an explicit `STATE` (`STATE_ON`, `STATE_OFF`, `STATE_TURN_ON`, `STATE_TURN_OFF`). `GetState()` is virtual and widely overridden.

- `env_state` – a simple entity that holds a toggleable state
- `multi_watcher` / `watcher` / `watcher_count` – watch the state of other entities
- `ShouldToggle()` updated to use `GetState()`

### USE Types Extended
Three new `USE_TYPE` values beyond the original set:

| Value | Meaning |
|-------|---------|
| `USE_SAME` | Re-fire with the same use-type as received |
| `USE_NOT` | Invert the use-type |
| `USE_KILL` | Kill the target entity |

### Think-Time Management
Reliable think scheduling in the presence of the MoveWith system:

- `SetNextThink(delta)` / `AbsoluteNextThink(time)` / `DontThink()`
- `m_fNextThink` + `m_fPevNextThink` shadow the engine's `pev->nextthink` to detect and correct drift
- `ThinkCorrection()` called every frame for all assisted entities

### Trigger Enhancements (`triggers.cpp`)
| Entity | Description |
|--------|-------------|
| `multi_manager` | Master key, random/sequential modes, thread names, locus thread, trigger-state control |
| `multi_watcher` | Watches the state of multiple entities and fires when all match |
| `trigger_inout` | Fires separate targets for enter/exit events |
| `trigger_bounce` | Reverses an entity's velocity on touch |
| `trigger_onsight` | Fires when one entity can see another |
| `trigger_startpatrol` | Sends a monster to start patrolling |
| `trigger_motion` | Applies motion (velocity/avelocity/force) to entities |
| `motion_manager` | Manages ongoing motion effects |
| `trigger_rottest` | (Development/test) rotational trigger |
| `trigger_changelevel` | Enforced lowercase map names to prevent engine issues |
| `trigger_teleport` | Landmark-based teleportation and angle-setting |
| `trigger_sound` | Plays a sound when triggered |
| `RenderFxManager` | Fades entity render colour in/out over time |

### Effects Enhancements (`effects.cpp`)
| Entity | Description |
|--------|-------------|
| `info_target` | Promoted to a full entity class, forced to use `null.spr` |
| `env_beam` / `env_laser` | Tripbeam support, healing beams, non-restriking mode, locus start/end points |
| `env_beamtrail` | Beam that follows an entity, leaving a trail |
| `env_rain` | Realistic rain/precipitation effect |
| `env_warpball` | Xen monster warp-in effect |
| `env_shockwave` | Expanding ring shockwave (like Houndeye attack) |
| `env_dlight` | Dynamic entity light (keyed, attached to entity) |
| `env_elight` | Dynamic entity light variant |
| `env_fade` | Added permanent-hold flag (`SF_FADE_PERMANENT`) |
| `env_shooter` | Added debug spawnflag |
| `env_particle` | Particle-system emitter |

### Lighting System (`lights.cpp`)
- `light_dynamic` – dynamic light that follows an entity
- `trigger_lightstyle` – temporarily overrides a light's style pattern
- `GetStdLightStyle()` utility for reading standard light styles
- Full `STATE` support on all light entities

### Train & Platform Enhancements (`plats.cpp`)
- `func_train`: backward-search mode for path corners, teleport-and-turn-to-face corners, non-crushing mode (`pev->dmg == -1`), rotation (`m_vecAvelocity`) support, movement sound persistence after save/load
- `func_tracktrain`: master angular-velocity (`m_vecMasterAvel`), base angular-velocity (`m_vecBaseAvel`), speed-proportional sounds
- `path_corner`: angle-facing support via `armorvalue` field
- `scripted_trainsequence` – scripted sequences for trains

### Scripted Sequence Enhancements (`scripted.cpp`)
- `scripted_action` – new entity alias for `CCineMonster`
- `aiscripted_sequence` – AI-driven scripted sequence (no separate class needed)
- `scripted_sentence` – triggers spoken NPC sentences with state tracking
- `scripted_tanksequence` – controls a `func_tank` through a scripted sequence
- Repeater count (`m_iRepeats` / `m_iRepeatsLeft`) on all scripted sequences
- Fire-on-begin targets
- Clean-up fix when a new script starts immediately after another

### Sound System (`sound.cpp`)
- `ambient_generic`: `play from entity` feature (plays the sound as if emitted by another entity on a specific channel)
- `trigger_sound` – triggers ambient sounds when entities enter/leave a volume

### Func_Rotating Enhancements (`bmodels.cpp`)
- Spin-up / spin-down speed ramp (`m_fCurSpeed`)
- Full `STATE` support
- `WaitForStart` to work around engine startup timing
- Shiny-surface support (`func_shine`)

### Door Enhancements (`doors.cpp`, `doors.h`)
- Immediate-mode doors (bypass normal open/close logic)
- Speed-override for MoveWith compatibility
- State-based close detection

### Button & Multisource Enhancements (`buttons.cpp`)
- `FCAP_ONLYDIRECT_USE` – button can only be used when directly in front (not through walls)
- `SF_BUTTON_ONLYDIRECT` spawnflag
- `multisource` redesigned: while `STATE_OFF`, mastered entities are completely blocked
- `env_spark` – full STATE support

### Func_Tank Enhancements (`func_tank.cpp`)
- Laser-spot targeting (`SF_TANK_LASERSPOT`)
- Match-target mode (`SF_TANK_MATCHTARGET`)
- Fire-master entity (gate for firing)
- Locus-based custom shot position (`m_pFireProxy`)
- `scripted_tanksequence` integration

### Monster Enhancements (`monsters.cpp`, `monsters.h`)
- `m_iClass` key – override monster class (supports `CLASS_FACTION_A`–`D` from `cbase.h`)
- `m_iPlayerReact` – fine-grained player-reaction control
- `SF_MONSTER_NO_YELLOW_BLOBS` – suppress "monster stuck" debug visualisation
- `SF_MONSTER_NO_WPN_DROP` – monster never drops its weapon
- `monster_target` – invisible target entity for monsters to shoot at

### Player Enhancements (`player.cpp`)
- New engine messages: `SetFog`, `KeyedDLight`, `SetSky`, `HUDColor`, `AddShine`, `Particle`
- `env_fog` – server-side fog control
- `env_sky` – runtime skybox replacement
- Direct-use logic: tries to find an exact entity in front of the player before falling back to standard use
- `FCAP_ONLYDIRECT_USE` respected during player use
- `USE_TOGGLE` / `USE_ON` / `USE_OFF` cheat commands (impulse 90–92)

### Utility Additions (`util.h`, `util.cpp`)
- `UTIL_AxisRotationToAngles` / `UTIL_AxisRotationToVec`
- `UTIL_StringToRandomVector` – parse randomised vector strings
- `UTIL_IsFacing` – check if an entity faces a reference point
- `SF_PUSH_NOPULL` / `SF_PUSH_USECUSTOMSIZE` spawnflags for `trigger_push`
- `GetStdLightStyle()` – read built-in light styles
- Global state mapping between `STATE` values and `GLOBAL_STATE` values (`util.h:187`)

### HUD & Client-Side (`cl_dll/`)
- `HUDColor` message handler – runtime HUD colour changes
- `hud_msg.cpp` extended with Spirit-specific message handlers
- `tri.cpp` – render additions for Spirit effects
- `ammo.cpp` / `ammo_secondary.cpp` – HUD ammo display updates
- `hl_baseentity.cpp` / `hl_weapons.cpp` – client-side weapon/entity updates
- `StudioModelRenderer.cpp` – renderer extensions
- Particle manager (`particlemgr.cpp`)

---

## New Entities Summary

| Category | Entities |
|----------|----------|
| Calculation | `calc_position`, `calc_ratio`, `calc_subvelocity`, `calc_velocity_path`, `calc_velocity_polar` |
| Aliases | `multi_alias`, `trigger_changealias` |
| State/Logic | `env_state`, `multi_watcher`, `watcher`, `watcher_count` |
| Effects | `env_beamtrail`, `env_customize`, `env_dlight`, `env_elight`, `env_footsteps`, `env_fog`, `env_model`, `env_particle`, `env_quakefx`, `env_rain`, `env_shockwave`, `env_sky`, `env_warpball` |
| Lighting | `light_dynamic`, `trigger_lightstyle` |
| Triggers | `motion_manager`, `trigger_bounce`, `trigger_changecvar`, `trigger_changevalue`, `trigger_inout`, `trigger_motion`, `trigger_onsight`, `trigger_rottest`, `trigger_sound`, `trigger_startpatrol` |
| Scripted | `aiscripted_sequence`, `scripted_action`, `scripted_tanksequence`, `scripted_trainsequence` |
| Monster | `monster_target` |
| Brush | `func_shine` |

---

## Building

The source code is in the `SpiritSource/` directory. It targets **Visual Studio 2019** (or later). See `SpiritSource/README.md` for full build instructions and differences from vanilla SoHL 1.2.

---

## License

The source code is provided under the [Half-Life 1 SDK License](SpiritSource/LICENSE). See `SpiritSource/README.md` for full license text.

