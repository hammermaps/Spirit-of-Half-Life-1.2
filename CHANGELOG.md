# Changelog – Spirit of Half-Life 1.2a

All changes relative to the vanilla Half-Life 1 SDK.  
Changes by **Laurie Cheers** (LRC) are documented below, grouped by source file / subsystem.  
They are identified in the code by `//LRC` comments.

---

## Core Entity System (`cbase.h`, `cbase.cpp`)

### New capability flag
- `FCAP_ONLYDIRECT_USE` (`0x00000100`) – entity cannot be used through a wall.

### New USE_TYPE values
- `USE_SAME` – re-fire using the same use-type as received.
- `USE_NOT` – invert the use-type before firing.
- `USE_KILL` – kill the target entity instead of toggling it.

### New entity classes (monster factions)
- `CLASS_FACTION_A` (14) – new faction class for use with "Behaves As" monster key.

### MoveWith fields added to `CBaseEntity`
- `m_MoveWith` – targetname of the entity to move with.
- `m_pMoveWith` / `m_pChildMoveWith` / `m_pSiblingMoveWith` – linked-list pointers for the MoveWith hierarchy.
- `m_vecMoveWithOffset` / `m_vecRotWithOffset` – positional and rotational offsets from parent.
- `m_activated` – tracks whether `Activate()` has been called.
- `m_iLFlags` – locus-related flags.
- `m_iStyle` – light-style index.
- `m_vecPostAssistVel` / `m_vecPostAssistAVel` – velocity state for the assist system.

### Think-time management
- `m_fNextThink` / `m_fPevNextThink` – shadow `pev->nextthink` to detect engine-side drift.
- `SetNextThink(delta)` / `AbsoluteNextThink(time)` / `DontThink()` – new convenience methods.
- `AbortThink()` – called by parent when a think must be cancelled.
- `ThinkCorrection()` – corrects `pev->nextthink` drift; called every frame.

### Locus / Alias virtual methods
- `CalcPosition()` / `CalcVelocity()` / `CalcRatio()` – virtual hooks for the locus system.
- `GetAliasTarget()` – virtual hook for the alias system.

### Activation
- `virtual Activate()` – called after all entities have spawned; used by all Spirit entities for deferred initialisation.
- `InitMoveWith()` – sets up MoveWith relationships during `Activate()`.
- `PostSpawn()` – optional entity-specific post-spawn logic (called by `Activate()`).

### Miscellaneous `CBaseEntity` changes
- `m_hActivator` moved from `CBaseToggle` to `CBaseEntity`.
- `ShouldToggle(USE_TYPE)` updated to use `GetState()`.
- Level-transition fix: entities that MoveWith something only cross if their parent crosses.

### `CBaseToggle` changes
- `LinearMoveNow()` / `LinearMoveDoneNow()` / `AngularMoveNow()` – think functions for guaranteed deferred linear/angular moves.
- `m_flAngularMoveSpeed` serialisation.
- Global-state / toggle-state mapping helpers.

### Global variables
- `g_pWorld` – pointer to the world entity, accessible everywhere.
- Alias-list and assist-list globals moved to `cbase.h` (from `alias.cpp`).

---

## MoveWith System (`movewith.cpp`, `movewith.h`)

- New subsystem: any entity can declare a `movewith` key and will follow its parent's position and rotation.
- Assist-list: tracks all entities that need per-frame velocity assistance.
- `UTIL_SetVelocity()` / `UTIL_SetAvelocity()` / `UTIL_SetAngles()` – wrappers that propagate changes through the MoveWith hierarchy.
- `DesiredAction()` / `DesiredThink()` – deferred think scheduling that works correctly under MoveWith.
- `g_doingDesired` flag prevents re-entrant desired-action processing.
- Assist-list and alias-list are rebuilt on game restore (`Activate()` pass).
- Simple `RotWith` helper for angular attachment.
- Limit on recursive MoveWith loops to detect infinite chains.

---

## Locus System (`locus.cpp`, `locus.h`)

- General-purpose position/velocity/ratio reference system.
- `$locus` keyword usable in entity key/value strings.
- `calc_position` entity – computes a world position from a named reference.
- `calc_velocity_path` / `calc_velocity_polar` – compute velocity vectors.
- `calc_ratio` – computes a numeric ratio from two named values.
- `calc_subvelocity` – sub-velocity calculation helper.
- `UTIL_StringToVector` extended with locus-aware parsing.

---

## Alias System (`alias.cpp`)

- `multi_alias` – maps a targetname to different real targets depending on the alias state.
- `trigger_changealias` – changes the alias target at runtime.
- Alias list rebuilt on `Activate()` after game restore.

---

## Trigger System (`triggers.cpp`)

### `multi_manager` enhancements
- `master` key – a master entity can block the manager.
- `m_flWait` / `m_flMaxWait` – constant and random minimum wait times between triggers.
- `m_iMode` – sequential (`0`), random-pick (`1`), each-random (`2`) modes.
- `m_iszThreadName` – rename the cloned manager thread for identification.
- `m_iszLocusThread` – locus reference passed through to cloned threads.
- `triggerstate` key – force a specific `USE_TYPE` on all fired targets.
- `SF_MULTIMAN_SAMETRIG` spawnflag – clone uses the same trigger type as parent.

### New entities
- `multi_watcher` – fires targets when all watched entities match a given state.
- `watcher` / `watcher_count` – helpers for `multi_watcher`.
- `trigger_inout` – fires separate `on_enter` / `on_exit` targets when entities enter/leave the volume.
- `trigger_bounce` – reverses the velocity of touching entities.
- `trigger_onsight` – fires when one entity has line-of-sight to another.
- `trigger_startpatrol` – orders a monster to begin patrolling.
- `trigger_motion` – applies continuous velocity, angular-velocity, or force to entities.
- `motion_manager` – manages ongoing motion effects started by `trigger_motion`.
- `trigger_rottest` – (development/test) rotation trigger.
- `trigger_sound` – plays a sound when entities touch the volume.

### `trigger_changelevel` fix
- Map names are forced to lowercase to prevent level-transition failures caused by mixed-case names.

### `trigger_teleport` enhancement
- Landmark-based teleportation (relative position/velocity preserved).
- Optional angle-setting when teleporting (`pev->v_angle`).

### `RenderFxManager` / `RenderFxFader`
- `env_render` extended with fade-in/fade-out capability.
- Handles `renderamt == 0` in normal mode by treating it as fully visible for fade purposes.

### `trigger_push` spawnflags
- `SF_PUSH_NOPULL` (512) – only push, never pull.
- `SF_PUSH_USECUSTOMSIZE` – use a custom size (reserved for future use).

---

## Effects System (`effects.cpp`, `effects.h`)

### `info_target`
- Promoted to a full `CPointEntity` subclass.
- Forced to use `null.spr` sprite model.

### `env_beam` / `env_laser` enhancements
- Tripbeam mode – beam fires when something breaks the line.
- Healing beam – damages or heals entities in contact.
- Non-restriking mode (`m_restrike == -1`).
- Locus start/end points.
- `BeamUpdatePoints()` – updates beam attachment each frame.
- `DamageThink()` renamed to a more general think function.

### `env_beamtrail`
- New entity: a beam that follows an entity and leaves a fading trail.

### `env_rain`
- New entity: long-awaited rain/precipitation effect using sprites.

### `env_warpball`
- New entity: Xen monster warp-in flash effect.

### `env_shockwave`
- New entity: expanding ring shockwave (similar to Houndeye attack).
- `m_iHeight` stores half the actual ring height (doubled when drawn).

### `env_dlight`
- New entity: dynamic entity light with a key name for client-side control.

### `env_elight`
- New entity: alternate dynamic entity light.

### `env_fade`
- `SF_FADE_PERMANENT` (`0x0008`) – hold the fade permanently (adds `FFADE_STAYOUT`).

### `env_shooter`
- `SF_GIBSHOOTER_DEBUG` (`4`) – debug spawnflag.

### Other
- `env_sprite` receives `USE_OFF` / `USE_ON` firing of its own targets when it turns off/on.

---

## Lighting System (`lights.cpp`)

- `light_dynamic` – dynamic light entity; follows an entity and updates each frame.
- `trigger_lightstyle` – temporarily overrides a named light's style pattern.
- `SetStyle()` / `SetCorrectStyle()` – methods to safely change light styles.
- Full `STATE` support (`GetState()`) on light entities.
- `GetStdLightStyle(int iStyle)` – utility to read built-in Half-Life light style patterns; declared in `util.h` for global access.

---

## Sound System (`sound.cpp`)

### `ambient_generic` enhancements
- `m_pPlayFrom` – play the sound as if it originates from another entity.
- `m_iChannel` – choose the sound channel.
- `StartPlayFrom()` think – deferred start to work around Activate-phase limitations.

### New entity
- `trigger_sound` – plays a looping ambient sound when an entity is inside the volume.

---

## Train & Platform System (`plats.cpp`)

### `func_train` enhancements
- Backward path-corner search mode.
- Teleport-and-turn-to-face corner (using `armorvalue` field on `path_corner`).
- Non-crushing mode: `pev->dmg == -1` prevents train from hurting players.
- Movement sound persistence across save/load (sound re-emitted on restore).
- Rotation support via `m_vecAvelocity`.
- `m_pSequence` pointer for `scripted_trainsequence` integration.

### `func_tracktrain` enhancements
- `m_vecMasterAvel` / `m_vecBaseAvel` – master and base angular velocity for curves.
- Speed-proportional sound system.
- `m_pSequence` for `scripted_tracktrainsequence`.

### `path_corner` enhancement
- `armorvalue` field: if non-zero, the train faces this angle when passing the corner.

### New entity
- `scripted_trainsequence` – scripted sequence that controls a `func_train` or `func_tracktrain`.

---

## Scripted Sequences (`scripted.cpp`, `scripted.h`)

- `scripted_action` – entity alias for `CCineMonster`; preferred name for action sequences.
- `aiscripted_sequence` – alias for AI-driven sequences (no separate class needed).
- `scripted_sentence` – triggers an NPC spoken sentence; has `m_playing` state.
- `scripted_tanksequence` – scripted sequence that drives a `func_tank`.
- Repeater: `m_iRepeats` / `m_iRepeatsLeft` on all scripted sequences.
- `m_iszFireOnBegin` – fire targets when the sequence begins.
- `m_iszAttack` / `m_iszMoveTarget` fields on `CCineMonster`.
- Clean-up so that if a new script starts immediately, the monster notices it.
- `CineThink()` called directly when appropriate.
- AMBUSH schedule selection fix.

---

## Player (`player.cpp`)

### New network messages registered
| Message | Size | Purpose |
|---------|------|---------|
| `SetFog` | 9 bytes | Server-side fog parameters |
| `KeyedDLight` | variable | Dynamic light with key |
| `SetSky` | 7 bytes | Runtime skybox replacement |
| `HUDColor` | 4 bytes | Runtime HUD colour change |
| `AddShine` | variable | Shiny-surface effect |
| `Particle` | variable | Particle effect |

### New entities driven from `player.cpp`
- `env_fog` – sends `SetFog` message to all clients.
- `env_sky` – sends `SetSky` message.
- `env_footsteps` – surface-based footstep sounds.

### Use logic
- Direct-use: tries to find an exact entity directly in front of the player before the standard trace.
- `FCAP_ONLYDIRECT_USE` respected in both direct and standard use paths.
- Impulse 90/91/92 cheat commands fire `USE_TOGGLE` / `USE_ON` / `USE_OFF` on the aimed entity.

### Miscellaneous
- Flatline sound suppressed when no suit is equipped (unless deathmatch).
- `g_markFrameBounds` debug flag.

---

## Button & Multisource System (`buttons.cpp`)

- `SF_BUTTON_ONLYDIRECT` (16) – button can only be used when the player is directly in front.
- `env_state` entity – simple toggleable state holder.
- `multisource` redesign – while `STATE_OFF`, all mastered entities are fully blocked.
- `env_spark` – full `STATE` support (`m_iState`, `STATE_ON` / `STATE_OFF`).
- Rotary button: boundary checking and corrected `avelocity` calculation.

---

## Door System (`doors.cpp`, `doors.h`)

- Immediate-mode doors (`-1` speed) that bypass normal door logic.
- MoveWith-compatible speed override.
- State-based close detection (instead of position-only check).
- `func_door_rotating` fixes for flags and immediate mode.

---

## Func_Rotating & Brush Models (`bmodels.cpp`)

- `func_rotating` spin-up/spin-down speed ramp (`m_fCurSpeed`, `m_flFanFriction`).
- Full `STATE` support on `func_rotating`.
- `WaitForStart()` think – deferred start to work around engine 1.1.0.8 startup timing.
- `func_shine` – brush entity with shiny-surface effect.
- Bug fix: `BModelOrigin()` now returns `(absmin + absmax) * 0.5` for rotating entities.

---

## Func_Tank (`func_tank.cpp`)

- `SF_TANK_LASERSPOT` (`0x0040`) – show a laser spot targeting reticle.
- `SF_TANK_MATCHTARGET` (`0x0080`) – match the visible target exactly.
- `SF_TANK_SEQFIRE` – internal flag: a `CTankSequence` is ordering the tank to fire.
- `m_iCrosshair` – show crosshair while player is using the tank.
- `m_pControls` – pointer to associated `func_tankcontrols`.
- `m_pSequence` / `m_pSequenceEnemy` – current sequence and its target.
- `m_iszFireMaster` – master entity that gates firing.
- `m_pFireProxy` / `m_iszLocusFire` – locus-based custom shot origin.

---

## Monster System (`monsters.cpp`, `monsters.h`)

- `m_iClass` key – override a monster's faction class at spawn time.
- `m_iPlayerReact` – fine-grained control over how a monster reacts to the player.
- `SF_MONSTER_NO_YELLOW_BLOBS` (128) – suppress "monster stuck" yellow debug blobs and log spam.
- `SF_MONSTER_NO_WPN_DROP` (1024) – monster never drops its weapon on death.
- `monster_target` – invisible entity that monsters can shoot at.
- `SpawnRandomGibs()` static helper on `CBaseMonster`.

---

## AI / NPC Schedules (`AI_BaseNPC_Schedule.cpp`, `defaultai.cpp`)

- `defaultai.cpp`: two LRC fixes to schedule selection and NPC reaction logic.
- `AI_BaseNPC_Schedule.cpp`: three LRC fixes to schedule running.

---

## Talk Monsters (`talkmonster.cpp`, `talkmonster.h`)

- 12 LRC changes: master checks, locus references, follow-entity improvements, state tracking.

---

## Combat System (`combat.cpp`)

- 10 LRC fixes: gib spawning helpers, damage propagation, locus-aware damage position.

---

## Utility Functions (`util.h`, `util.cpp`)

- `UTIL_AxisRotationToAngles(vec, angle)` – convert axis+angle to Euler angles.
- `UTIL_AxisRotationToVec(vec, angle)` – convert axis+angle to a direction vector.
- `UTIL_StringToRandomVector(pVector, pString)` – parse a vector string with random range support.
- `UTIL_IsFacing(pevTest, reference)` – check if an entity is facing a reference point.
- `GetStdLightStyle(iStyle)` – return the standard Half-Life light style string for a given index.
- Global states mapping (`STATE` ↔ `GLOBAL_STATE`) documented in `util.h:187`.
- `UTIL_DesiredAction()` exported for use by trigger entities.

---

## HUD & Client DLL (`cl_dll/`)

- `hud.cpp` / `hud.h` – 12 + 11 LRC changes: `HUDColor` message, Spirit-specific HUD elements.
- `hud_msg.cpp` – 9 LRC changes: new message handlers for `SetFog`, `KeyedDLight`, `SetSky`, `HUDColor`, `Particle`.
- `hud_redraw.cpp` – 2 LRC changes: HUD redraw updates.
- `health.cpp` – 3 LRC changes: health display updates.
- `ammo.cpp` / `ammo_secondary.cpp` – HUD ammo display changes.
- `tri.cpp` – 4 LRC changes: rendering additions for Spirit effects.
- `view.cpp` – 4 LRC changes: client-side view effects.
- `hl/hl_baseentity.cpp` – 7 LRC changes: client entity base updates.
- `hl/hl_weapons.cpp` – 3 LRC changes: client weapon updates.
- `StudioModelRenderer.cpp` – 2 LRC changes: model rendering extensions.
- `particlemgr.cpp` – particle manager for `env_particle`.

---

## Common / Engine Headers (`common/`, `engine/`)

- `common/const.h` – 4 LRC additions: new render flag constants used by Spirit effects.
- `engine/eiface.h` – 1 LRC change: engine interface update.
- `dlls/cdll_dll.h` / `cl_dll/cl_dll.h` – 1 LRC change each: message-ID declarations.

---

## Game Rules (`gamerules.cpp`, `gamerules.h`, `singleplay_gamerules.cpp`)

- Minor LRC fixes to support new USE_TYPE values and state checks.

---

## Miscellaneous Files

| File | LRC Changes |
|------|-------------|
| `animation.cpp` / `animation.h` | 2 + 2 – animation sequence fixes |
| `barnacle.cpp` | 4 – barnacle grab/release fixes |
| `barney.cpp` | 8 – Barney follower + sentence improvements |
| `bigmomma.cpp` | 3 – Big Mama attack fixes |
| `bloater.cpp` | 2 – Bloater NPC fixes |
| `bullsquid.cpp` | 3 – Bullsquid attack fixes |
| `client.cpp` | 4 – client connection hooks |
| `controller.cpp` | 2 – Controller NPC fixes |
| `crowbar.cpp` | 1 – crowbar use fix |
| `explode.cpp` | 2 – explosion locus support |
| `func_break.cpp` / `func_break.h` | 10 + 1 – breakable enhancements |
| `game.cpp` / `game.h` | 4 + 1 – game rule cvar additions |
| `gargantua.cpp` | 2 – Gargantua fixes |
| `genericmonster.cpp` | 5 – generic monster improvements |
| `gman.cpp` | 2 – G-Man sequence fix |
| `h_battery.cpp` | 3 – battery charger state |
| `hassassin.cpp` | 5 – assassin NPC fixes |
| `headcrab.cpp` | 4 – headcrab attack fixes |
| `healthkit.cpp` | 3 – health kit state |
| `hgrunt.cpp` | 8 – HECU grunt improvements |
| `houndeye.cpp` | 2 – Houndeye attack fix |
| `ichthyosaur.cpp` | 2 – Ichthyosaur fix |
| `islave.cpp` / `islave_deamon.cpp` | 3 + 3 – Alien Slave fixes |
| `items.cpp` | 4 – item pickup improvements |
| `leech.cpp` | 2 – Leech NPC fix |
| `maprules.cpp` | 2 – map rule fixes |
| `monstermaker.cpp` | 3 – MonsterMaker locus support |
| `nihilanth.cpp` | 2 – Nihilanth fight fix |
| `osprey.cpp` | 8 – Osprey sequence improvements |
| `pathcorner.cpp` | 2 – path corner angle support |
| `rat.cpp` | 3 – Rat NPC fix |
| `roach.cpp` | 2 – Roach NPC fix |
| `rpg.cpp` | 1 – RPG laser spot fix |
| `scientist.cpp` | 5 – Scientist sentence improvements |
| `schedule.h` | 1 – schedule state check |
| `subs.cpp` | 13 – `SUB_UseTargets` MoveWith integration |
| `tempmonster.cpp` | 1 – temporary monster fix |
| `trains.h` | 10 – train state/flag additions |
| `turret.cpp` | 7 – turret state and master support |
| `weapons.cpp` / `weapons.h` | 1 + 2 – weapon fire locus support |
| `world.cpp` | 6 – worldspawn Spirit keys |
| `zombie.cpp` | 2 – Zombie attack fix |

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| Files with `//LRC` comments | ~90 |
| Total `//LRC` occurrences | ~590 |
| New Spirit-specific entities | ~35 |
| New Spirit-specific source files | 6 (`locus.cpp/h`, `movewith.cpp/h`, `alias.cpp`, `ambient_2d.cpp`, `ambient_mp3.cpp`, `playermonster.cpp`) |
