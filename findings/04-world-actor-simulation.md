# FFXI PS2 Client (SCUS_972.66) — World & Actor Simulation

Decompiled retail PS2 client, build `version 972 (sha256=d4622c1cc4a3012c, image_base=0x00100000)`,
Ghidra 12.0.4 output with recovered DWARF symbols. Scope: movement, collision, zoning, entity
management, lifts/doors. Goal is to surface rules/constants that a server emulator (LandSandBoat /
"LSB") must agree with.

**Confidence legend**
- **[CODE]** — value or rule is literally present in the decompiled source at the cited line.
- **[DWARF]** — backed by a recovered field/type name (struct member, function signature).
- **[RODATA]** — referenced only by a `DAT_0054cXXX` rodata load address; the *role* is clear from
  use, but the literal byte value is NOT in this decomp artifact and must be read from the binary's
  `.rodata` at the cited address. Treat the numeric value as unknown unless re-extracted.
- **[INFER]** — interpretation/derivation, flagged as such.

> **Big caveat for tuning floats.** Most movement tuning constants in `xicontrolactor.cpp` are
> compiled as rodata loads (`DAT_0054c208`, `DAT_0054c20c`, …). The decomp emits only the *address*,
> not the float. The values that ARE exact are inline immediates (and hex-float immediates, decoded
> below). Anything marked [RODATA] needs a second pass against the ELF `.rodata` to get the number.

---

## A. HARDEST CONSTANTS FIRST (spatial / timing) and the ENTITY-ID SCHEME

### A1. Entity / target ID scheme — PC vs NPC/mob discriminator = high byte `0xFF000000`  **[CODE]**
The single most server-relevant rule. The client distinguishes players from NPC/mobs purely by the
top byte of the 32-bit server id (`atel_work->id.UniqueNo`):

- `GetTargetNearestPC` — `xiactor.cpp:1483`: candidate is a **PC** when `(UniqueNo & 0xff000000) == 0`.
- `GetTargetNearestNPC` — `xiactor.cpp:1580`: candidate is an **NPC/mob** when `(UniqueNo & 0xff000000) != 0`.
- `GetId` — `xiactor.cpp:676`: returns `this->atel_work->id.UniqueNo` (full 32-bit server id), else 0.

**Server relevance (high):** This is exactly FFXI's id layout that LSB implements — PC ids are "low"
(zone-encoded server id with zero high byte in the client's view), NPC/mob ids carry a non-zero high
byte. Any id LSB sends to this client MUST keep that invariant or the client will mis-bucket PCs and
NPCs in target cycling. Note: the **targid (0–1023) array and index↔id translation are NOT in this
base-actor layer** — `XiActor` manages actors as an intrusive doubly-linked list (`_top__7XiActor`,
`Link`/`Unlink`/`GetHead`/`GetNext`), with no fixed `actor[1024]` array and no max-actor count. The
targid table lives in the `XiAtelBuff` / atel layer (the struct that owns `id.UniqueNo`, `ActorInfo`,
`game_status`), which is outside the files in this scope.

### A2. Offline/debug actor id = live actor count  **[CODE]**
`XiActor::Init` (`xiactor.cpp:294-295`): when `is_playing==0 || is_netconnected==0`, sets
`this->id = GetActorsNum()` (the current count walked from the list). Confirms there is no real id
pool in this layer; online ids come entirely from the server. **Server relevance:** none directly,
but confirms the client trusts server-assigned ids exclusively when connected.

### A3. Targeting / acquisition radius = 50.0 units (2500.0 squared)  **[CODE]**
- `GetTargetAuto` (`xiactor.cpp:1074`): max auto-target squared distance `2500.0` → 50.0 units.
- `GetTargetLeft`/`GetTargetRight` (`xiactor.cpp:1281,1393`): `if (dist2 < 2500.0)` gate → 50.0 units.
- `GetTargetNearestPC`/`NearestNPC` (`xiactor.cpp:1470,1567`): `fStack_2c = 2500.0` squared cap;
  uses real `sqrt` for the actual distance.
- `GetTargetLeft`/`Right` initial angular score bound `3600.0` (`xiactor.cpp:1258,1370`).
- `CheckTargetDistance` (`xiactor.cpp:1660`): compares `dx²+dy²+dz²` to a caller-supplied distance;
  also rejects when `target_flag (offset 0x91) == 2`.

**Server relevance (medium):** 50.0-unit client-side target reach. LSB's "too far away" / target
validation should not be stricter than this for actions the client can initiate by cursor.

### A4. Sub-actor activation radius = 6.0 units (36.0 squared)  **[CODE]**
`XiActor::SysMove` (`xiactor.cpp:535`): `fVar8 = 36.0` squared threshold selecting the nearest actor
for sub-actor processing → 6.0 units. **Server relevance (low/informational):** a proximity LOD,
not a sync rule.

### A5. Movement is normalized per 60 fps; hard speed cap 30.0; status-5 doubles; back = 0.25x  **[CODE]**
From `XiControlActor::StepControl` (`xicontrolactor.cpp`):
- Line ~638: `dist = speed / 60.0;` — per-frame distance is speed divided by **60.0** (60 fps).
- Lines ~635-637: `if (EnableSpeedLimit && 30.0 < speed) speed = 30.0;` — **hard cap 30.0 units/s**.
- Lines ~631-634: in `game_status == 5` the speed is **doubled** (`speed * 2.0`).
- Lines ~838, 933: backward movement (move-vec Y<0) scaled by `0x3e800000` = **0.25** (25% speed).
- `ChangeVectorLengthByDirection` (~1376-1395): forward uses the actor base speed getter
  (vtable `+0xdc`) `/60.0`, `*2.0` in status 5; strafe/back uses `DAT_0054c20c` [RODATA].

**Server relevance (high):** LSB sends speed as a byte (FFXI movement-speed field). The client's hard
ceiling of 30.0 units/s and the per-frame /60 normalization define what visible speed the client can
produce; the status-5 (likely sneak/special-state) doubling and the 0.25x backwalk are client-side
visuals the server should not contradict in its own movement validation/speedhack checks.

### A6. Heading is RADIANS, atan2-derived, sign-flipped — NOT a 0-255 byte  **[CODE] / [DWARF]**
The `dir` field (vector at struct `+0xc0`, working copy `+0x20`; X-component = heading) is an angle in
**radians**:
- `StepControl` (`xicontrolactor.cpp:1011`): `dir = -atan2f(dz, dx);` (note the **negation**).
- Same pattern at `~1001-1005`, `CircleMove ~1329-1339`, `ApproachControl ~2736-2737`,
  auto-run `~761-765` (`atan2f(run_z, run_x)`).
- `OnMove` (`~253-306`): the three dir components are **angle-wrapped** with `fmodf` against the
  ±π / ±2π rodata constants `DAT_0054c260` (≈2π), `DAT_0054c264` (≈π), `DAT_0054c27c`, etc. [RODATA].
- `GetDirId_Div8` (`~1524`): `angle * DAT_0054c818` then bucketed against literal **45.0, 90.0,
  135.0, 180.0** (`~1525-1532`) — `DAT_0054c818` is the radians→degrees factor (≈57.2958 = 180/π)
  [INFER from the 45/90/135/180 comparisons]. Produces an 8-way compass direction id.

**Corroborated independently in `XiSkeletonActor`** (the model/NPC/mob actor): heading is a float
Euler **vec3 in radians** at struct `+0x290` (`dir`=yaw, `+0x294`=pitch, `+0x298`=roll; `pos` vec3 at
`+0x280`). There is **no `0x43800000`, no `/256.0`, no `& 0xff`** anywhere in `xisklactor.cpp`.
- `SetDir` (`xisklactor.h:76-146`): wraps each component with `fmodf` + add/sub-2π (`DAT_0054ca00`,
  `DAT_0054c950`, `DAT_0054ca0c`) — classic radian normalization to ±π. [CODE]
- Passed to the renderer as radians: `KzObject::SetRot(obj, &dir)` (`xisklactor.cpp:1546`),
  `sceVu0RotMatrix(..., &dir)` (`xisklactor.cpp:1601`). [CODE]
- Target facing: `facing = -atan2f(velZ, velX)` (`xisklactor.cpp:1121-1122`) — same sign-flip. [CODE]

**Server relevance (high):** The server stores heading as a 0-255 byte (`rotation`, FFXI convention);
the client engine uses radians internally — the byte↔radian conversion happens in the **network
layer**, not in these actor classes. Conversion the client implies:
`byte_dir = round( -radians * 256 / (2*PI) ) & 0xFF` (note the sign flip; equivalently
`byte = rotation * 128/PI`). LSB and any client must agree on the sign convention or characters face
backwards. The `GetDirId_Div8` 8-way bucketing is the client's coarse facing used for some
interactions/animations.

### A7. Door/lift collision-rect "occupied" sync uses `KO_RectData.flag` and rect-hit tests  **[CODE]/[DWARF]**
World objects (doors, elevators) are `KO_RectData` rectangles in the zone resource. They are NOT pure
visuals; they gate collision:
- `XiDoorActor::SetDoor` (`xidooractor.cpp:334-335`): on bind sets `this->rect->flag = 1` (closed/solid).
- `OpenDoor` (`xidooractor.cpp:638-639`): if opened with `EnableCollision==0`, sets `rect->flag = 0`
  (passable). `CloseDoor` (`xidooractor.cpp:719-720`) sets `rect->flag = 1` again.
- Player collision reads these rects: `XiCollisionActor::OnMove` (`xicollisionactor.cpp:307`)
  `LiftRectHitCk(pos_in, pos_out)` — if the player is inside a lift rect, the player is **snapped to
  the lift** (`is_on_lift = 1`, ground material from `(rect+0x30 >> 4) & 0xf`, ground normal set to a
  fixed up-vector at `xicollisionactor.cpp:316-320`).

**Server relevance (high):** door open/close state and elevator membership are world state the server
drives. LSB must broadcast door/lift state transitions so the client's `rect->flag` (collision
solidity) and `is_on_lift` stay correct; otherwise players clip through closed doors or fall off
moving lifts.

### A8. Position precision — lift floor heights are 16-bit fixed-point, 1/256 unit  **[CODE]**
`XiLiftActor::SetLift` (`xiliftactor.cpp:227-229`):
```
HeightTbl       = (float)(short)rect->lift_height  / 256.0 + rect->y;
field_0x19c     = (float)(short)rect->field_0x36   / 256.0 + rect->y;
```
Floor heights are signed 16-bit offsets from the rect base Y, scaled by **1/256** (≈0.0039 units
precision, ±128-unit range). **Server relevance (medium):** if LSB tracks lift floor positions, this
is the encoding/precision; the client cannot represent lift offsets finer than 1/256 unit.

---

## B. LIFTS / ELEVATORS — timing & state sync

Lift motion is **scheduler-driven**, not free-running on a client timer. `XiLiftActor::LiftMove`
(`xiliftactor.cpp:411`) and `SetLiftSchedule` (`xiliftactor.cpp:507`) build a resource id of the form
`"mv" + fromFloor + toFloor` (the `0x7600` / `0x6d` = `"vm"` little-endian + ASCII floor digits
`'0'+index`, `xiliftactor.cpp:441-442`, `537`) and run it through `YmScheduler::Execute`. The schedule
total time comes from the resource (`YmScheduler::total_frame`), so **timing is data-driven from the
zone DAT, not a hardcoded constant.**

- `XiLiftActor::OnMove` (`xiliftactor.cpp:116-125`): each frame copies `field_0xb0` → pos and sets
  W=`0x3f800000` (1.0); the lift's actual Y comes from `SetLiftHeight`.
- `SetLiftHeight` (`xiliftactor.cpp:336`): writes `rect->lift_current_height` and rewrites every part
  matrix — this is the live elevator height shared with the collision system via the rect.
- Lift floor selection: `ResIDToDstFloor` (`xiliftactor.cpp:391-395`) decodes destination floor from
  the resource id high byte minus `0x30` (ASCII '0').

**Server relevance (high):** elevator schedules run on a fixed cadence defined by the zone resource
file; both client and server read the same DAT, so LSB must drive lift floor changes on the same
named-schedule cadence (FFXI elevators move on real-time minute boundaries). The client trusts the
resource timing — desync means players board/exit at the wrong height.

---

## C. DOORS — timing & state

- **Door open/close animation length is data-driven**, taken from the scheduler resource:
  `InitOpenDoor`/`OpenDoor`/`CloseDoor`/`InitCloseDoor` all call `YmScheduler::CalcTotalFrame` then
  `SchTotalTime = total_frame` (`xidooractor.cpp:457-459, 542-543, 627-629, 713-715`). The countdown
  runs in `OnMove` (`xidooractor.cpp:139-145`) decremented by `iRam00557a44` (frame-time tick).
- **One hardcoded door timing constant:** `XiDoorActor::ActivateEntModel` (`xidooractor.cpp:788`)
  sets the spawned `KO_EntAnmTask` field to `0x3c` = **60** (frames = 1.0 s at 60 fps) — the
  entity/animation activation delay for door-attached models. **[CODE]**
- Resource ids are FourCC: open = `"open"`/`"evst"`, close = `"clos"`/`"eved"`, init-open `"otin"`/
  `"stin"`, init-close `"ctin"`/`"etin"` (`xidooractor.cpp:413-417, 501-505, 588-593, 676-681`) —
  the `door != NULL` branch picks the model-door variant, else the simple-door variant.
- Door angle wrapping: `SetDoorAngle` (`xidooractor.cpp:868`) wraps each Euler component with `fmodf`
  against `DAT_0054c82c..0054c85c` (the ±π/±2π family, same as movement) [RODATA].
- Door slide scale: `MakeDoorMatrix` (`xidooractor.cpp:994`) scales the slide vector by
  `0x3d800000` = **0.0625** (1/16) [CODE] — a units→matrix slide factor.

**Server relevance (high):** doors open/close on server command; the visible animation duration is in
the zone DAT (matched by both sides). LSB needs to trigger the right open/close action and respect
that `rect->flag` toggles collision (see A7) — the door is solid while closed.

---

## D. COLLISION

- **Per-actor collision radius = `DAT_0054c244 * sqrt(bboxX² + bboxZ²)`** where the bbox half-extents
  come from actor attribute `0x2c` via vtable `+0xf4`. `XiControlActor::GetCollisionSize`
  (`xicontrolactor.cpp:3506-3517`). The scale `DAT_0054c244` is [RODATA] (value not in artifact).
- **Two-actor contact test:** `(distance - sizeA - sizeB) < 0` (`xicontrolactor.cpp:819, 1153, 1239,
  1652`). `CheckContactActor` (`~1614`) seeds nearest-distance with `64.0` [CODE] (search window),
  and on contact sets a **30-frame** (`0x1e`) contact timer (`~1655`) [CODE].
- **World/terrain collision** is delegated to `KO_*` map functions, not done in these classes:
  `KO_CharaCollision`, `KO_CharaCollisionFast`, `KO_GetGroundStatus`, `GetCurrentGroundHeight`,
  `GetCurrentWaterHeight`, `GetCurrentPlace` (indoor flag), `GetCurrentAreaID`, `GetCurrentPlight`
  — all called from `XiCollisionActor::OnMove` (`xicollisionactor.cpp:238, 248, 267, 279, 284-296`).
- **Hit-check throttling / dead-reckoning of collision:** `XiCollisionActor` runs a staggered hit
  check. Constructor (`xicollisionactor.cpp:56-57`) seeds `hit_check_count = rand() % 0x78` (0..119).
  `OnMove` decrements by the frame tick and, when expired, resets to `0x78` = **120** frames
  (`xicollisionactor.cpp:185-194`) — i.e. a full collision re-poll roughly every **2 seconds @ 60fps**
  when far from geometry. **[CODE]**
- **Movement-quantum collision gate:** `OnMove` only runs the expensive `KO_CharaCollision` when the
  per-frame move exceeds `DAT_0054c1fc` and the actor moved into a new cell; otherwise it reuses
  `last_hit_chk_result` (a cached resolved position) — a client-side optimization
  (`xicollisionactor.cpp:232-245`). The `last_hit_chk_pos` is initialized to `1e+09`
  (`xicollisionactor.cpp:54`) as an "invalid/far" sentinel. **[CODE]**
- **"Close enough to do collision at all" gate:** `OnMove` (`xicollisionactor.cpp:202`)
  `bVar1 = (dz² + dx² + dy²) <= 10000.0` against the camera/ref point `iRam00557a58+0x50` →
  collision is only computed within **100.0 units** of that reference. **[CODE]**
- **Ground normal correction:** `CorrectNormal` (`xicollisionactor.cpp:74`) samples ground at two
  half-step points (scale `0x3f000000` = 0.5) and blends the slope normal. Slope acceptance in
  `OnMove` (`xicollisionactor.cpp:280`): `if (DAT_0054c248 < ny || ny < DAT_0054c2b8)` set ground
  normal [RODATA] — the slope-limit thresholds are in rodata.

**Server relevance (medium):** the client does its own terrain/actor collision and may *reuse a cached
resolved position* (`last_hit_chk_result`) rather than re-resolving every frame. This means reported
positions are the client's collision-resolved positions, polled on a staggered ~2 s cadence when
stationary, full-rate when moving. LSB position validation should expect client-resolved (snapped)
positions, not raw input positions, and should tolerate the quantization from the move-quantum gate.

---

## E. MOVEMENT (player control) — additional constants

(From `XiControlActor`; see A5/A6 for the headline rules.)
- **Turn / yaw rate:** `DAT_0054c208 * frame_tick`, used as `*2.0*stick` for analog turn
  (`xicontrolactor.cpp:628, 737, 826, 920`) [RODATA].
- **"Is moving" threshold:** `OnMove` (`~222`) `is_moving = 1` when per-frame displacement
  > `DAT_0054c26c` [RODATA].
- **Near-zero epsilon:** `DAT_0054c1f4` (`~239-240`, also `SetPos ~59`, `OnKeyDown ~423`) — minimum
  meaningful displacement [RODATA].
- **Walk vs run blend:** analog partial-tilt sets `mot_weight = DAT_0054c1ec`; deadzone
  `DAT_0054c1b0`, full-tilt `DAT_0054c1ac` (`AdjustAnalogKeyLength ~1055-1079`) [RODATA].
- **Auto-run break:** dot-product threshold `-0.5` (inline) cancels auto-run when input opposes
  heading (`StepControl ~793`) [CODE].
- **Approach / arrival distance:** `DAT_0054c1fc` (`StepControl ~668, 716`; `ApproachControl ~2716,
  2803`) [RODATA].
- **Ground-plane projection (no climbing up surfaces):** `StepControl ~937-955` projects the move
  vector onto the ground plane (double cross-product with ground normal) and **clamps upward Y to 0**
  (`if (0.0 < vec.y) vec.y = 0;`, `~946`) — downhill-only correction. **[CODE]**

### Gravity / jump / fall  **[CODE]**
- **No ballistic physics in normal play.** `v_velocity` (struct `+0xd0`) is only ever set to 0.0
  (constructor `~1743`, `StepControl ~1026`).
- **Jump exists only as a GM debug feature:** gated by `gcConfGmLevelGet() > 3`
  (`StepControl ~1024-1029`): sets `v_velocity = 0`, drops Y by `1.0`, and `_DebugJumpCount = 0x78`
  (**120** frames). Confirms FFXI has no player jump; vertical position is governed by ground-follow,
  not jumping. **Server relevance (high, as a negative):** LSB should never receive/expect jump
  velocity from this client.

### E2. Skeletal actor: speed→animation rate, walk/run thresholds, state machine  **[CODE]/[RODATA]**
From `XiSkeletonActor` (`xisklactor.cpp`) — the model actor used for players/NPCs/mobs. This is how
the **server-synced base speed** drives the visible movement animation; LSB's speed byte directly
feeds the expected-distance calc here.

- **Server-synced speed lives at field `+0xd4`** (read via vtable `+0xd4`), and the *current* anim/move
  speed multiplier at `+0xcc`. `GetMoveVectorRatio` (`xisklactor.cpp:1792`) computes
  `expectedDist = (serverSpeed / 60.0) * frameCount` (reference FPS **60.0**), with chocobo
  (GameStatus 5) doubling it `(speed*2.0)/60.0` (`xisklactor.cpp:1822-1828`). [CODE] — matches the
  control-actor /60 + status-5 doubling (§A5), confirming the rule from a second class.
- **Animation ratio = distMoved / expectedDist, smoothed `ratio*0.25 + prev*0.75`, clamped to max 3.0**
  (`xisklactor.cpp:1836-1855`). Stored to `move_vec_ratio` (`+0x3b8`). [CODE]
- **Walk vs Run cutoff = 1.5** (`SelectMotionResId`, `xisklactor.cpp:1934-1947`): if
  `ratio/DAT_0054c1ec > 1.5` → RUN motion (`field_0x34c`="run"), else WALK (`field_0x348`="wlk");
  below `DAT_0054c238` → idle. `DAT_0054c1ec` = walk-speed unit normalizer, `DAT_0054c238` =
  idle→walk distance threshold [RODATA]. The **1.5 cutoff is an inline literal** [CODE].
- **Movement-state machine keyed on `XiActor::GetGameStatus`** (motion FourCC tags, little-endian):
  idle `idl`, walk `wlk`, run `run`, battle-stance `btl`, strafe `mvb`/`mvr`/`mvl`, corpse/dead
  `cor`, sit `si1`, rest `rx1`, heal/sit `fh1`. Notable status values: `2,3`=corpse/dead pose, `4`=KO,
  `5`=chocobo (mounted, speed×2), `6`=sit/heal, `7`/`0x21`=rest, `0x26-0x2b`=fishing, `0x2f`=sit.
  (`xisklactor.cpp:780-783, 1387-1393, 1918-1972`). [CODE] **Server relevance (high):** these
  game-status values are the same status field LSB sets; the client picks animation purely from it, so
  status semantics must match (esp. 5=chocobo triggering the speed doubling).
- **Heading interpolation (smooth turn) = 0.125 (1/8) per substep** toward target each frame
  (`OnMove`, `xisklactor.cpp:1039-1047, 1153-1161`), looped `frame_delta` times, shortest-path wrapped.
  [CODE] **Server relevance (medium):** the client visually slerps to the server-sent heading; an
  instantaneous server heading change is smoothed over ~8 substeps client-side (affects what heading
  the client *appears* at vs the authoritative value).
- **Footstep audio range = 30 units (900.0 squared)** — `PlayFootSteps` (`xisklactor.cpp:7898`)
  `if (dist² < 900.0)`. [CODE] (proximity audio LOD, not a sync rule).
- Default `mot_speed = 1.0` (`+0x358`), `scale = 1.0` (`+0x2c4`), `velocity = 0.0`,
  `blinkeye_timer = 120.0`, idle-transition blend = **16.0 frames** (`xisklactor.cpp:765,776,785,1717`).
  Alpha fade-in `+= frames*0.03125` (1/32), fade-out `-= frames*0.0625` (1/16)
  (`xisklactor.cpp:1477-1516`). [CODE]
- Model scale/collision: visible scale via `KzObject::SetScale`; width/height/depth =
  `bbox_dim*scale / DAT_0054c2bc|c2c0|c2c4` (`xisklactor.cpp:5968/6006/6044`) [RODATA]; the actual
  collision radius comes from `XiControlActor::GetCollisionSize` (§D), not stored in the skel actor.

### Knockback / blowback  **[CODE]/[RODATA]**
- `blowback_dumper` default `0.125` (constructor `~1781`) — per-step decay of knockback velocity.
- `BlowBackParaTab` (`StepControl ~3167`): an **8-entry × 12-byte** table `{power, dumper, timer}`
  indexed by `scale_no` 0–7 (knockback magnitude classes). Values are in rodata, not in the artifact.

---

## F. ZONING

The cross-zone teleport itself is server-driven and lives in the `gc*` glue / YK-UI layer (out of
scope files). What `src/main/zone/` and the collision layer contain:

- **Indoor sub-map change trigger:** `XiZone::OpenIndoorWait`/`OpenIndoor`
  (`xizone.cpp:302, 359`) call `gcZoneSubMapChangeSet(2, map_num & 0xffff)` — the client tells the
  server (op `2`) it is entering indoor sub-map `map_num`. Indoor maps are resource id `map_num + 100`
  (`xizone.cpp:308, 368`). **[CODE]** **Server relevance (high):** sub-area (indoor/instance-ish)
  transitions are reported with this call; LSB's zone server tracks which sub-map a player is in.
- **Zone load sequence:** `XiZone::Open` → `OpenByDvdPos` (`xizone.cpp:104, 128`): closes current
  zone (`Close` → `XiActor::KillAll`, `xizone.cpp:560`), loads the new zone resource, resets camera
  (`YmCameraManager::InitPos`), sets place code 0, sets current area. `SetCameraInitTimer`
  (`xizone.cpp:110`) re-inits the camera on zone-in. **[CODE]**
- **Area object is a linked tree** (`XiArea::GetHead/Link/Unlink/Find`, `xiarea.cpp:1243+`); a zone is
  the root `XiArea`. `xiarea.cpp` is overwhelmingly **weather + lighting + fog + clip-range**, not
  zone-line geometry. The drawn world clip range is `XiArea::GetClipRange` → `current_world->clip_range`
  (`xiarea.cpp:474-478`) — a per-world DAT value, not a constant. **[CODE]/[DWARF]**
- **No zone-line trigger distance constant exists in the client.** There is no client-side
  "you crossed the line, change zones" geometry here — zone transitions are commanded by the server.
  The closest client-side spatial gating is the `KO_RectData` system (doors/lifts, §A7) and the
  indoor sub-map change above. **[INFER]** Marked uncertain: do not assume a client zone-line radius.

**Server relevance (high):** zoning is authoritative on the server. The client reports indoor sub-map
entry via `gcZoneSubMapChangeSet`; everything else (which zone, spawn position) is pushed by the
server, and the client clears all actors (`KillAll`) and re-inits camera on each zone change.

---

## G. ACTOR LIFECYCLE / LISTS (for completeness)

- Actors are an **intrusive doubly-linked list** rooted at `_top__7XiActor`
  (`xiactor.cpp:89, 109, 138, 195`); traversal `GetHead/GetNext/GetPrev/GetTail`
  (`xiactor.cpp:2700, 2732, 2764, 2670`) skips `XiDollActor` instances. `GetActorsNum`
  (`xiactor.cpp:457`) counts by walking — **no fixed-size actor array, no MAX_ACTORS in this layer.**
- `KillAll` (`xiactor.cpp:18`) destroys every actor (used on zone close).
- `FindActor(char*)` (`xiactor.cpp:1914`) finds by **name string only** — there is **no
  FindActor-by-id / GetActor(targid)** in the base class; id→actor lookup is in the atel layer.
- Default actor scale `0x40800000` = **4.0**; default move speed = `debug_actor_move_speed * 4.0`
  (`xiactor.cpp:280-282`); default name color `0x80808080` (`xiactor.cpp:249`). **[CODE]**
- `IsChocobo`: `0 < subactor_status < 5` (`xiactor.h:1896`); `IsFishingRod`: `subactor_status > 4`
  (`xiactor.h:1918`) — sub-actor state ranges. **[CODE]**

---

## H. NAME / SCREEN-SPACE (not entity ids — disambiguation)

`XiActor::NameCalc` (`xiactor.cpp:855`) and `GetTargetScreen` (`xiactor.cpp:1154`) use a screen-space
fixed-point centered at `2048.0` (`pos*resolution*0.5 + 2048.0`, `xiactor.cpp:1154-1158`); name
culling compares against `0x1000` (4096) and `0xf001` (61441) guard-band coordinates
(`xiactor.cpp:912-919`). These are **screen pixel coordinates, NOT id masks or world distances** —
called out to avoid mis-reading the `0x1000`/`0xf001` literals as id ranges.

---

## TOP-10 SUMMARY (server-relevance ordered)

1. **PC vs NPC/mob = `id & 0xFF000000`** (==0 player, !=0 NPC/mob). `xiactor.cpp:1483/1580`. [CODE]
2. **Heading = radians, `dir = -atan2f(dz,dx)`, sign-flipped, wrapped ±π/±2π** (confirmed in BOTH
   `xicontrolactor` and `xisklactor`; no byte/256 encoding anywhere in the engine — byte↔radian is in
   the net layer). Convert to server 0-255 byte: `byte = (-radians)*128/PI & 0xFF`.
   `xicontrolactor.cpp:1011`, `xisklactor.cpp:1121-1122`, `SetDir xisklactor.h:76-146`;
   8-way bucket via `GetDirId_Div8 ~1524`. [CODE]
3. **Movement: /60 per-frame, hard cap 30.0 units/s, status-5 (chocobo) doubles, backwalk 0.25x;
   walk→run animation cutoff at ratio 1.5.** `xicontrolactor.cpp:~635-638, 838`,
   `xisklactor.cpp:1822-1828, 1934-1947`. [CODE] (the /60 + status-5 doubling appears in BOTH the
   control actor and the skeletal actor — strongly corroborated.)
4. **No player jump / no gravity** — `v_velocity` always 0; jump is GM-debug only.
   `xicontrolactor.cpp:1024-1029`. [CODE]
5. **Targeting reach = 50.0 units (2500.0²)** across all target-pick functions. `xiactor.cpp:1074,
   1281, 1470`. [CODE]
6. **Doors/lifts gate collision via `KO_RectData.flag`** (1=solid, 0=passable); player snaps to lift
   rects (`is_on_lift`). `xidooractor.cpp:638-639/719-720`, `xicollisionactor.cpp:307-322`. [CODE]
7. **Lift/door timing is data-driven from zone DAT scheduler `total_frame`**, not hardcoded; one fixed
   door entity-anim delay = 60 frames (`0x3c`). `xiliftactor.cpp:485`, `xidooractor.cpp:457/788`. [CODE]
8. **Lift floor heights are 16-bit fixed-point at 1/256-unit precision** off the rect base Y.
   `xiliftactor.cpp:227-229`. [CODE]
9. **Client reports collision-resolved (snapped) positions, polled on a staggered ~120-frame (~2 s)
   cadence when stationary / full-rate when moving**, reusing a cached resolved position between polls;
   collision only computed within 100.0 units (10000.0²) of the reference point.
   `xicollisionactor.cpp:185-194, 202, 232-245`. [CODE]
10. **Indoor sub-map entry is reported to the server via `gcZoneSubMapChangeSet(2, map&0xffff)`;**
    cross-zone teleport is server-authoritative; on zone change client `KillAll`s actors and re-inits
    camera. `xizone.cpp:302/359, 110, 560`. [CODE]

### Open items needing a `.rodata` extraction pass (values not in this decomp)
Turn rate `DAT_0054c208`, strafe speed `DAT_0054c20c`, collision-radius scale `DAT_0054c244`,
"is moving" threshold `DAT_0054c26c`, walk/run blend `DAT_0054c1ec`/`1ac`/`1b0`, slope limits
`DAT_0054c248`/`0054c2b8`, approach distance `DAT_0054c1fc`, the ±π/±2π wrap family
(`DAT_0054c260/264/27c`, door `0054c82c..85c`), radians→degrees `DAT_0054c818`, and `BlowBackParaTab`.
All cited by address above; re-read from `SCUS_972.66` `.rodata` at the listed offsets to recover the
literal floats.
