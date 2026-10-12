# L4D2 Pose Parameter Lag Compensation Fix (SourceMod)

Makes Left 4 Dead 2's lag compensated hitboxes match what the shooter's client drew: position, rotation, pose parameters and animation frame of survivors, special infected, common infected and witches.

## Requirements

- SourceMod 1.11 or newer (DHooks and SDKHooks are built in).
- Windows: `server.dll` md5 `533888fbb4e5ed534b172470613a3017`.
- Linux: `server_srv.so` md5 `bf54b537f4707f0c602208c427ec62df` (dedicated server) and `server.so` md5 `e962508781885d7b0d871d79e456b03d` (listen server). The Linux signatures match once in both files.
- Every platform-dependent value (signatures, member offsets, the `CAnimationLayer` layout) is in the gamedata. The plugin itself is the same on both systems. Points 9 to 14 each need their own gamedata entries. If one is missing, only that point is off and the error log says which.
- Compatible with `l4d2_lagcomp_manager`, which detours the same game function.

## Install

- `addons/sourcemod/plugins/l4d2_posesync.smx`
- `addons/sourcemod/gamedata/l4d2_posesync.games.txt`
- `addons/sourcemod/scripting/l4d2_posesync.sp` (source)

The first start writes `cfg/sourcemod/l4d2_posesync.cfg` with the four cvars below. If you ran an earlier build, delete the old cfg once so it is written again without the removed cvars.

## What it corrects

The engine's lag compensation rewinds origin and angles to the shooter's target time (newest record - lerp - latency allowance; the engine already accounts for ping, and nothing is added to it). Sequence, cycle and overlay layers are only rewound when the caller asks for it, and gunshots never get it (point 10). Pose parameters are never rewound. The client draws other players and infected differently, and the plugin makes the rewound server skeleton match what the client drew.

### 1. Pose parameters

Pose parameters are recorded every tick and rewound with the engine, which never stores them.

### 2. Players' body yaw

`SetupBones` builds player bones with the animstate's render yaw, not the entity angles, so it is recorded and rewound too.

### 3. Eye-driven values

`C_TerrorPlayer` interpolates other players' `m_angEyeAngles` like their origin (client.dll registers `C_CSPlayer::m_iv_angEyeAngles` for it), and its animstate aims and turns with that value:

- `body_yaw`, `head_yaw` and the feet yaw come from the time the client draws.
- `body_pitch` and `head_pitch` are rebuilt from the networked eye pitch (`m_angEyeAngles[0]`) the way the client eases them (`ComputePoseParam_AimPitch`, server_srv.so 0x5001f0): each frame `body_pitch` moves 12 x dt of the way to the eye pitch and `head_pitch` 12 x dt of the way to what is left, over the records of the last 0.6 s. The server's own `body_pitch` follows its animstate's eye pitch, which can be far from the networked one.
- Eye lock: during a shove stagger, a pounce knock-off, a Charger pummel or pound, or a wall / ground slam, the client's animstate stops taking new eye angles (client.dll 0x1027eb30), so the body keeps the aim and turn it had when that started, while the server's keeps following the eyes. The plugin takes the eye values from the time the client drew when the stagger started. The client's goal feet yaw is that held eye yaw and its feet keep turning toward it (`ConvergeAngles`, below; anim_showstate during a shove showed the feet go from -166.9 to the held -129.0 within half a second, with `body_yaw` = eye yaw - feet yaw following), so the feet yaw and `body_yaw` are turned from their values at the start for the time since. A stagger can outlast the lag compensation history (a 1.5 s stagger at 30 tick); the eye angles and render yaw of the half second before every eye-lock start are kept per player for that.
- Turning after a spawn. The animstate's feet goal, turn rate and aim easing depend on its speed (server_srv.so `UpdateFootYaw` 0x504260, `ConvergeAngles` 0x503020, `ComputePoseParam_AimYaw` 0x5004f0; the same code runs in client.dll):
  - goal: the direction of travel once the speed is over 0.9 x the ground speed (turned around when that is more than 100 degrees from the eyes), otherwise unchanged; then kept within 45 degrees of the eye yaw (90 for survivors).
  - feet: min(180 r x clamp(gap / 45, 0.01, 1), 100 r) degrees/s toward the goal (`mp_feetyawrate` 180, `mp_feetmaxyawrate` 100), r = 2 above 0.5 u/s and 1 below: 200 degrees/s down to a 25 degree gap while moving, then the gap shrinks by e^-8 per second.
  - aim: `body_yaw` eases toward eye yaw - feet yaw at 10 per second while moving and 6 standing, `head_yaw` takes the rest at 20 per second.
- The client's speed is its estimate: 0 for the reset window after it gets a spawned player (point 7), then the estimate from the snapshots it got (point 14), which trails the true speed. So in the window its feet turn at half the server's rate and its `body_yaw` eases more slowly, and afterwards its feet goal switches to the direction of travel only when its estimate passes 90% of the ground speed, later than the server's, and toward the direction of its estimate. The plugin runs this for both sides over the records since the reset (the server with its true velocity, the client with the rebuilt estimate at each tick) and adds the client's difference to the server's feet yaw, `body_yaw` and `head_yaw`, for up to 2 s.
- When the window ends while the player moves, the client mostly draws its newest snapshot, undelayed, for one frame, and its estimate jumps far past any threshold, which points the feet goal at the movement at once: anim_showstate showed 1150, 559 and 2615 u/s in three of four spawns (the drawn position leaping ahead for that frame and back on the next). In the fourth (200 ms of latency, as in one of the three) the estimate rose smoothly (50, 63, 112, 142, 158) and the goal switched only at 157.96. The server cannot see which frame the client draws, so the jump is assumed; when it does not happen, the client's feet stay near eye yaw + 45 until its estimate passes 90% and the hitboxes are turned too far until then.
- Checked against anim_showstate of three spawned Jockeys: the model of the server's feet stayed within 1.5 degrees of the server's over the first second, and the client's feet followed the same rules frame by frame. In the jump cases, turning to the run direction at 200 degrees/s: -0.2, -3.8, -13.9, -23.7, -33.8, -43.8, -53.4 while the server's crept toward eye yaw + 45. In the smooth case, held near eye yaw + 45 at -10.3, -12.1, -13.2, -14.4, -16.2 while the server's turned to -37.5.
- A new entity. When the client makes the player entity (it reached the shooter for the first time since either connected, as with a spawned bot, which can be sent from its connect, before the first record of its life), its pose parameters start at 0 normalized, the bottom of their range, while the server's start at 0 degrees, and its feet start at the eye yaw of the first state it got (-10.0 in anim_showstate, where the server's started at 0). anim_showstate of a spawned Jockey: `body_yaw` -87.1 and `body_pitch` -84.2 one frame in on the client, 0 and 15.2 on the server. They then ease as above: `body_yaw` at 6 per second while standing, so about 20 degrees of the difference is left a quarter second after the spawn; the model of the client's `body_yaw` stayed within 0.2 degrees of the client's from the spawn to the shot. The plugin starts the client's `body_yaw` / `head_yaw` (point above) and `body_pitch` / `head_pitch` there in that case. A player coming back into view after going dormant keeps its values.

### 4. Animation frame from the client clock

Target + lerp + 1 tick. The client plays animations on its own clock with the newest data it has; only the position is drawn behind. Its clock is one tick ahead of the `tick_count` its command carries.

#### Players

- Main sequence, cycle and overlay layers.
- Movement animations (idle, walk, run, crouch and the 3x3 move blends) are played on the client's clock like every other sequence.
- The client restarts the main sequence at cycle 0 when a new `m_mainSequenceStartTime` (or a new `m_nSequence`, except between two activities of the same group) reaches it (0x1027e760, 0x10146320 -> 0x10145f10), at its own clock of that moment, and plays it on from there. anim_showstate prints `m_mainSequenceStartTime` after the cycle: the server time the sequence was last restarted, not a clock, so it stays the same for a whole run. The plugin records it per tick, ends a run where it changes, and takes the client's restart from the first snapshot that carried the change to the shooter (`SDKHook_SetTransmit`). The cycles the server played before that are worked out once per run and kept for the whole run, also after its first record has left the history.
- The client plays a run at the cycle rate of its own move blend, set from its speed estimate, which trails the true speed while the player speeds up (and is 0 in the reset window after a special infected spawns, item 7). That is not added: counting it only worked while the run's start was still in the history (about the first second at a 200 ms round trip), and in game those shots were the ones out of step, while later ones matched.
- The Hunter's pounce (`Pounce_01`): the client starts it when it arrives and restarts it its interpolation later; from then on it is handled like a movement animation.
- The crossfade from one sequence to the next is rebuilt on the same clock; a fading Hunter leap keeps its interpolation delay.

#### Common infected and witches

`(clock - m_sequenceStartTime) * cycle rate`, the way client.dll computes it. Further rules from client.dll's `C_Infected` (0x103146c4, run when a new start time arrives):

- A common that comes into a client's view with a start time new to it: if the sequence started less than 1 s before and the common had been out of view for over 1 s, the client starts it at cycle 0 on arrival. Otherwise it sets the frame from the start time at the model's cycle rate, without the playback rate (0.9 - 1.1 per entity for alert idles, speed based while moving), and plays on with the playback rate; the longer the sequence had been running, the further off that is, until the next sequence. The plugin follows what each common was sent to each client (`SDKHook_SetTransmit`) and builds the shooter's frame the same way.
- The same rule runs for a common in view when its sequence changes: the client starts the new sequence at cycle 0 when the snapshot with the new start time reaches it, not at the start time itself, so the plugin uses the time that snapshot was sent to the shooter.
- Repeated entries. Until the client's acknowledgement of the snapshot that brought a common into its view gets back to the server (about one round trip), each further snapshot is a delta from an older acknowledged one that did not have the common, so the common is sent as entering the view again. The client decodes it from the baseline each time, the start time proxies run again, and the sequence and the overlays start over from that snapshot. Measured with `cl_showanimstate` on a freshly spawned common: the last restart came 0, about 0.1 and about 0.2 s after the first snapshot at `net_fakelag` 0, 50 and 100 (round trips of 0, 99 and 199 ms). The plugin reads the `tick_count` of each client's commands (its clock, the snapshot time it has reached) and treats every snapshot sent before one at or past the entry arrives as a new entry.
- `ACT_TERROR_ATTACK_CONTINUOUSLY`, `_LOW_CONTINUOUSLY`, `_DOOR_CONTINUOUSLY` and `ACT_TERROR_REACH_THROUGH_DOOR` start at a random 0 - 0.3 of the cycle on the client and at 0 on the server. The value is never sent, so the plugin adds the expected 0.15.
- The server plays at most 0.2 s of animation per update (`StudioFrameAdvance`). When a common's NextBot updates are further apart (`nb_update_framelimit`, busy servers), its own frame falls behind the client's for good. Cycle rates are measured over the time actually played.
- A common's simulation time only moves forward when it moves or turns (`CBaseEntity::SetLocalOrigin` / `SetLocalAngles`). One standing still keeps its old time, so the engine makes no lag record, while its NextBot goes on changing sequences (idle, alert, acquire, ...). The client gets every change in the next snapshot and plays it from `m_sequenceStartTime`. The plugin records such a change at the present time when `m_sequenceStartTime` changes without a new simulation time (or its overlay list changes), so the rewind uses the sequence the client is playing instead of the one from the last time the common moved. Without this, the server's hitboxes of a standing common that had just become alert could be in a completely different pose.
- Commons with a stopped simulation time still play on the client from the start time. A non-looping sequence that has finished restarts on the client, so it wraps.
- `BacktrackEntity` also returns false when there is nothing to move (the record at the target time is where the entity already is). The pose and frame corrections are kept in that case.
- Overlay layers. `DT_Infected` does not send a common's overlay layers either; it sends `m_clientAnimatedOverlay`, a copy of the first 6 (sequence, order, looping, and the server time the layer started), made on each NextBot update. The client plays each one itself in its slot `i` with order `i`: cycle = (its clock - start time) x the sequence's cycle rate (a finished non-looping one is switched off), weight 1 faded in and out over the sequence's fade times. Attack swings while a common closes in, and flinches, are such layers. The server's own layers have their own cycles and weights, and a common that has stopped to attack makes no lag record, so its swing was missing from the rewound skeleton. The plugin records the list (also when it changes without a new simulation time) and sets overlay slots 0 - 5 the client's way during the rewind.
- The client does not keep the server's start time of an overlay: its `m_flStartTime` proxy (0x10313340) replaces it with the client's own clock when the new value arrives, unless the common went dormant on that client within the last second. The plugin notes, per client and slot, the snapshot that first carried each start time and uses that time instead.
- Aim. A common's pose parameters are not networked; the client computes them itself (`C_Infected`, client.dll 0x10313c60). Every frame it turns `body_yaw` and `body_pitch` from the common's origin (its view offset is never sent, so it is 0 on the client) towards the eyes of its look-at target (`m_clientLookatTarget`) where the client draws that target: origin + 62 units for other players, the shooter's own eyes for the shooter. With no target both are 0. `lean_pitch` is the centre of its range. The server sets `body_yaw` and `body_pitch` only when its AI calls `AimHeadTowards`, from its own eye height at where the target was at that moment, and sets `lean_pitch` from its movement. So the torso and head of a common attacking a survivor who moves or crouches around it, or of one that has just picked a new look-at target (the client snaps the upper body round to it at once), were turned differently on the server. The plugin computes the three values the client's way from the rewound common and the target as the shooter's client drew it. `lean_yaw` comes from `m_clientLeanYaw`, which the client has as of its newest update, so it is taken from there instead of from the engine's target time.

### 5. Bullet broad phase

Players and common infected use rotation-expanded sequence bounds, so the box a bullet ray must cross before any hitbox is tested comes from the current sequence. When the plugin puts in the client's sequence, that box is rebuilt for it (`CCollisionProperty::MarkSurroundingBoundsDirty`), and again when it is put back.

### 6. Sequence crossfade

For about 0.2 s after a sequence change the client blends the old sequence out (min(fadeout, fadein), spline weight); the server never does. It is rebuilt from the recorded history as temporary overlay layers.

### 7. Interpolation reset window

When the client gets a player new (a spawn in view, a materialized special infected, a player back in its snapshots after 0.2 s without one) or the player gets `EF_NOINTERP` (a teleport), it draws the newest position without delay, with a 0 speed estimate (neutral move blend), for about its extrapolation limit (`cl_extrapolate_amount`, 0.25 s). Shots in that window are rewound to that position with `move_x` / `move_y` at 0.

The window counts from the reset, in the time the client draws, whether or not the player moves: with anim_showstate at 100 tick the estimate turned on 17 frames at 60 fps after the client first had a spawned Jockey, at lerp 20 ms and about 100 ms alike, so the plugin uses 0.275 s. A Jockey that stood 0.23 s before running had a 0 estimate for only the first 4 frames of the run; counting the window from the first movement (as before) kept shots 0.25 s into the run inside it after the client had left it.

Entering a shooter's snapshots is tracked per shooter from the snapshots it was sent. The frame is the one the rest of the run uses: taking the slower centre animation's cycle for the window put the start of a run behind the client in game.

### 8. Position and rotation time

The client draws other entities `TICKS_TO_TIME(TIME_TO_TICKS(lerp) + 1)` behind its clock, so its interpolation is whole ticks, and its command's `tick_count` is one tick behind that clock. What it draws when it fires is therefore `tick_count - round(lerp / tick interval)` ticks. The engine rewinds to `tick_count - lerp` (the exact lerp), so origin and angles are moved by `lerp - round(lerp / tick interval)` ticks.

| Lerp at 30 tick | Correction |
|---|---|
| 9 to 16 ms | +9 to 16 ms |
| 20 ms | -13 ms |
| 33.3, 66.7 or 100 ms | 0 |

The animation frame (point 4) is not changed by this.

### Gaps in what the engine rewinds at all

None of these is fixed by another plugin in SirPlease/L4D2-Competitive-Rework (`l4d2_lagcomp_manager` only adds a forward on the same game function, `l4d2_jockey_hitbox_fix` fixes the riding Jockey's IK height and the ridden survivor's bounding box, `l4d_rock_lagcomp` compensates the Tank's rock).

### 9. Swings

`CTerrorPlayer::WantsLagCompensationOnEntity` rewinds nothing unless the command holds attack or shove, or is within 5 commands of the last shove. A melee swing, a shove, a special infected's claw and the Tank's punch go on testing for hits for 0.2 s or more after the click, so a tapped melee swing is compensated only while the button is down, and the rest of it hits the present hitboxes. At 100 tick the shove allowance (5 commands) covers 50 ms of a 200 ms swing. While the active weapon is swinging (`m_bInMeleeSwing`, or the `m_swingTimer` of shoves, claws and the Tank's fist), the command counts as an attack for that test. The buttons are changed only for the length of that call.

### 10. Animation rewind for gunshots

`FireTerrorBullets` starts lag compensation with type "hitboxes along ray" but passes the world origin, a zero angle and range 0. `BacktrackEntity` rewinds the animation only of entities within 100 units of that ray, so for bullets the sequence, cycle, overlay layers and crouch bounds stay at the present. The plugin turns that call into the plain "hitboxes" type, which rewinds them for every entity. The plugin already puts in the client's sequence and cycle; what this adds is the overlay layers of common infected and witches (flinches, gestures) and the player crouch bounds.

### 11. Jockey on a survivor

A riding Jockey is parented to the survivor (`CBaseEntity::SetParent`) and the game never rewinds it: moving a child with `SetAbsOrigin` while its parent is rewound or put back would corrupt its offset. The survivor is not rewound either when the shooter is a survivor (teammates are skipped without `mp_friendlyfire`).

The shooter's client draws the Jockey attached to the survivor at the survivor's interpolated position, so the server's Jockey was ahead by the survivor's movement over lerp + ping. The plugin rewinds the ridden survivor for shooters on the other team than the Jockey; the Jockey moves with it, and nothing is set on the Jockey itself. Its bone cache is flushed after the rewind and again after everything is put back. A survivor carried by a Charger is attached the same way and already moves with the rewound Charger; its bone cache is flushed too.

### 12. Records more than 64 units apart

`BacktrackEntity` walks the records from the newest back and leaves the entity at the present when two neighbouring records (or the newest one and the present origin) are more than 64 units apart. Records are only made when the simulation time moves: common infected every NextBot update, players when their commands arrive. A common whose updates are late on a busy server (250 units/s needs 0.26 s), or a lagging player whose commands arrive in a burst, leaves such a gap without teleporting, and the client interpolates across it.

When every gap between the newest record and the target time is at most 1000 units/s, the model did not change, no interpolation reset happened and the entity has no parent, the plugin moves it to its recorded position (and, for common infected and witches, yaw) itself and puts it back afterwards. A real teleport (spawn, rescue closet, director teleport) is thousands of units/s and is left alone.

`SetAbsOrigin` / `SetAbsAngles` stamp the entity's simulation time with the current time. The plugin puts the old simulation time back with the position, or every client would take it as a new position sample and draw the common stalling and catching up on each shot.

### 13. Where the client draws a player or a common

Between the states it received. The client's origin is a `CInterpolatedVar` without `INTERPOLATE_LINEAR_ONLY`, so client.dll (0x1004aa10) draws it on a Hermite spline through the two samples around the draw time and the one before them (0x10048240, the older sample moved to the same spacing first), and linearly only when it has two.

Its samples are the states it received: a common's every NextBot update (about 0.1 s), a player's every snapshot the shooter was sent (the plugin follows these per client with `SDKHook_SetTransmit` on players; with `cl_updaterate` under the tick rate, or with choke, they are several ticks apart). Past its newest sample it holds it: it extrapolates only a value from the very latest packet, which a common's update never is at that point. With a lerp under 0.1 s the client often draws a common past its newest update.

The engine instead interpolates linearly between every record, including ones the client never received. The plugin builds the client's samples, puts the entity where the client drew it (curve, held sample or sparse snapshots) and puts the engine's position back before `FinishLagCompensation`. A moved player's eye angles come from the same samples.

### 14. Move blend from the client's speed estimate

The client gets velocity from `EstimateAbsVelocity` (0x1004c230), which is the derivative of the same samples (`GetDerivative_SmoothVelocity`, 0x10049780): the Hermite derivative with three, the newer segment with two, and 0 when it draws past its newest sample. The server uses the true velocity. Everything the client builds from it is rebuilt from the client's samples:

- Players (`CTerrorPlayerAnimState`, 0x10281e50, outside `ACT_TERROR_JOCKEY_RIDE`): `move_x` = cos(a) * f, `move_y` = sin(a) * f, with a = feet yaw - direction of travel and f = speed / ground speed clamped to 0.01 - 1 (0 under 0.5 u/s). The ground speed is the same on both sides and is taken from the server's own move blend and velocity in the records. The ground speed is `GetCurrentMaxGroundSpeed` (0x1027e270), which is 0 for every activity outside the idle, walk, run, crouch walk and jump families (shoves, slams, knock-offs, climbs, falls, crouch idles, ...: 66 activities, found by running client.dll's activity mapping 0x1027de60 over all of them). The client's move blend is then 0 even while the player still slides.
- The Hunter in a pounce, a pounce idle or a fall after a Charger hit (0x10282711): `vert_vel` and `horiz_vel` from the vertical and horizontal speed. These change every tick in the air, where the client's estimate trails the true velocity.
- Common infected (`C_Infected`, 0x10313c60): with fewer than 20 `C_Infected` on the client, the 9-way blend (0x10031190), which rescales `move_x` / `move_y` until the run animation's own speed (read from the model's movement data) matches the estimate, up to 6 passes within 512 units of the camera and one beyond. With 20 or more, the estimate divided by the run animation's ground speed (at that count the client's `move_y` has the opposite sign of the server's for sideways movement, and the plugin copies that too). The server divides its true velocity by the run speed on each NextBot update.
- An animation whose last movement block ends before its last frame keeps that block's end position as its motion, as `Studio_AnimPosition` (0x10342a10) does; many idles are made like that, so on the client even a standing common's `move_x` / `move_y` can go to the end of their range.

### Not changed

- Hitbox 16 of the common infected models (below). Moving it to the head group would change damage, not lag compensation.
- The feet yaw's turn in steps: the client converges it a little every rendered frame and the server every tick, so the client's depends on its frame rate. With the same speed both follow the same rule (point 3) and anim_showstate showed them within a degree or two at the same moment of play, so the plugin keeps the server's feet yaw at the time the client drew, apart from the spawn window.
- The Jockey's move blend while riding (`ACT_TERROR_JOCKEY_RIDE`). The client smooths it every rendered frame, for the same reason.
- Ladder climbing (dismount pose and climb speed), which the client works out per frame from the ladder.

## Performance

Recording runs for every lag compensated entity every tick, so it does no name lookups: member offsets are resolved once, each entity's class and its `m_sequenceStartTime` offset are cached per entity (players are classified again only after a spawn, death or team change), overlay layers are read through one cached vector address, and entities whose simulation time did not move (common infected between their 0.1 s updates) are skipped by comparing with a cached time. For common infected one more value is read first (the sequence start time, see point 4).

Rewinding (every nearby entity, every tick of a shove or melee swing) reads cvars from cached values, walks the history on the time cell of each record and reads only the one or two records it needs, uses per-sequence flags worked out when the model is read instead of comparing activity names, looks up each model's pose parameter indices once, and reads the shooter's interpolation straight from the engine. A player's run restart delay is worked out once per run.

The turn after a spawn (point 3) is only modelled for players whose newest reset is less than 2 s old, kept per player when it is recorded, so other players cost one comparison. For those within 2 s it rebuilds the client's speed estimate at each tick after the window. `body_pitch` reads the eye pitch of the last 0.6 s of records (60 at 100 tick) per rewound player.

What each client was sent (commons' visibility and start times, players' snapshots) is only tracked for human players in the first 32 slots. A common's overlay start times are read once per tick, not once per client it is sent to, and a client's copy is only compared again when one of them changed. The debug events are only hooked while `sm_posesync_debug` is on. `sm_posesync_profile` measures it on your server.

## Notes

Common infected are lag compensated by the engine. They update every 0.1 s, so there is one record per 3 ticks at 30 tick and the engine interpolates between them. `sv_showimpacts` never draws their hitboxes; `sm_posesync_drawhitboxes` does.

Hitbox 16 of every full common infected model is on the head bone but in hit group 3 (stomach): the jaw/mouth box at the front of the face.

## ConVars

| ConVar | Default | Description |
|---|---|---|
| `sm_posesync_enable` | 1 | On/off. |
| `sm_posesync_targets` | 15 | Bitmask: 1 survivors, 2 special infected, 4 common infected and witch, 8 anything else lag compensated. |
| `sm_posesync_debug` | 0 | 1 = per lag compensated player command: the lines of the entity it hit, then "this command hit ent N, hitbox H (group G)" (or "shoved ent N"). A gun shot that misses shows the entity your aim ray passed closest to; a shove, melee swing or claw that hits nothing prints nothing. 2 = also every changed pose parameter; 3 = also the full animation state and bone positions. |
| `sm_posesync_drawhitboxes` | 0 | 1 = draw the server's rewound hitboxes for the shooter (see below). |

Every correction in points 1 to 13 is always on. To compare with the game's own lag compensation, set `sm_posesync_enable 0`. The drawn hitboxes stay 4 seconds, like the game's `sv_showimpacts` boxes.

## Reading sm_posesync_debug 1

Per shot, for the entity that was hit:

- `Cycles (Server)`: the engine's cycle and animation at its target time.
- `Cycles (Client)`: the client's cycle and animation, and how it was computed.
- `Server - client`: the difference, in cycles and in ms of animation.
- `Calculation`: how the frame the hitboxes use was chosen.
- `Cycles (Updated server value)`: the cycle and animation the hitboxes are built with.

Players also get the position time, the body yaw, the `body_pitch` taken from the eye angles, and an "eye lock" line while the client holds its eye angles.

`position: your client draws it at T ...` says how the client drew it (Hermite spline through, linear between, or held at its samples, and whether they came from the snapshots you were sent) and how far that is from the engine's linear interpolation. It only appears when they differ by more than 0.05 units.

`start time ... was first sent to your client in the snapshot of T` (commons) is when the server first sent the sequence's start time to the shooter. `client overlay N: ... (started T when it reached you; server S, ...)` means the overlay was started at the time that snapshot was sent.

`move blend: your client's speed estimate ...` gives the client's speed, the `move_x` / `move_y` the hitboxes are built with and the server's own values; for commons also which of the client's two ways was used and how many commons your client had. Hunters in a pounce get a `pounce blend` line with `vert_vel` / `horiz_vel`.

`... s after a spawn / interpolation reset: your client turned with a 0 speed estimate ...` gives the feet yaw, `body_yaw` and `head_yaw` the client's turn after a spawn (point 3) changed, and the model's feet yaw for the server next to the server's recorded one, which shows how closely the model follows the server.

## Drawing the server's hitboxes (works on a dedicated server)

`sm_posesync_drawhitboxes 1` draws, when you shoot, shove or swing, for the entity your aim ray hits (or passes closest to), the hitboxes the hit was traced against, after every correction: green = hitboxes, yellow = the box your aim ray hits, white = your aim ray missed them all. Your console gets one line per snapshot, at most one snapshot every 0.5 s. Only entities the plugin rewound are candidates, so with `sm_posesync_enable 0` nothing is drawn. If many players draw at once, a new snapshot is skipped while about 2 s of beams are still waiting to be sent.

The beams arrive one round trip after the shot (the server learns of the shot when your command arrives, then the beams travel back), but they show the moment of the shot: compare them with the frame where you fired. On a listen server the blue `sv_showlagcompensation` boxes are drawn inside the engine's rewind, before the plugin's frame corrections, so use `sv_showimpacts` or these boxes instead. The aim ray ignores spread and punch.

The boxes cannot be drawn earlier: the server cannot send anything before your shot reaches it, and rolling the drawn boxes back by lerp + ping would show a state older than the one the bullet was traced against.

## Commands (root admin)

| Command | Description |
|---|---|
| `sm_posesync_status` | Counters, including how often each of points 9 to 14 applied. |
| `sm_posesync_dump <ent>` | The entity's present animation state. |
| `sm_posesync_history <ent> [n]` | Last n stored records. |
| `sm_posesync_trace <ent> [seconds]` | One line per server tick (simulation time, origin, speed, no-interp resets, sequence, cycle, yaw). 0 stops it. |
| `sm_posesync_profile [seconds]` | Measure the plugin's own CPU time (default 10 s): microseconds per server tick for recording, and per rewound entity for shots. |
| `sm_posesync_unhide` | Unhide the developer-only animstate debug cvars (`sv_showanimstate`, `cl_showanimstate`, ...). They are also unhidden at every map start. |
