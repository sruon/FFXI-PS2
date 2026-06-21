# Combat & Action Mechanics — Client-Side Constants and Rules

Source: decompiled FFXI PS2 client SCUS_972.66 (version 972, image_base 0x00100000),
Ghidra-style C++ with recovered DWARF symbols. Scope: `src/main/actor/`.

## How to read this document

- The decompiler recovered **function symbol names** (the `// symbol:` headers and method
  names like `XiActor::CheckTargetDistance`) — these are **authoritative**. Local variable
  names (`iStack_30`, `fVar4`) and many struct field names (`field_0x124`) are **inferred /
  generic**; where I interpret them I say so.
- Inline numeric literals (`2500.0`, `36.0`, `0x1ff`) are **real values from the binary**.
- `DAT_0054xxxx` / `BlowBackParaTab` / `GA_WAZA_COMBO_EFFECT` are **.rodata addresses**. The
  *address* is authoritative but the decomp source does **not** contain the float/byte value
  at that address, so I can only describe how it is used, not quote the number. These are
  flagged "(value in .rodata, not in source)".
- "Server relevance" = why an LSB / emulator developer should care.

---

# TIER 1 — Hard spatial limits, caps, and magic numbers

## 1. 50-yalm target-selection / target-validity range  (squared = 2500.0)

The classic FFXI "you can't target/keep something more than 50 yalms away" rule is baked in
as the literal `2500.0` (= 50.0²) used as the initial "max distance" in every nearest/screen
target scan, and explicitly as `50.0` passed to the target-length check.

- `src/main/actor/xiactor.cpp:1074` `GetTargetAuto`: `fVar10 = 2500.0;` (best-distance init)
- `src/main/actor/xiactor.cpp:1281` `GetTargetLeft`: `if (fVar8 < 2500.0)` (candidate gate)
- `src/main/actor/xiactor.cpp:1393` `GetTargetRight`: `if (fVar8 < 2500.0)`
- `src/main/actor/xiactor.cpp:1470` `GetTargetNearestPC`: `fStack_2c = 2500.0;`
- `src/main/actor/xiactor.cpp:1567` `GetTargetNearestNPC`: `fStack_2c = 2500.0;`
- `src/main/tk/TkInputCtrl.cpp:2004` `CKaTarget::CheckTargetLength(this,50.0);` — caller passes
  the literal **50.0**, which `CheckTargetLength` (TkInputCtrl.cpp:2257) squares to `len*len`
  and feeds to `XiActor::CheckTargetDistance(...)`.

`CheckTargetDistance` (xiactor.cpp:1660-1687) computes squared euclidean distance with all 3
axes and compares to the squared threshold; it ALSO returns 0 (invalid) if the target's state
byte `pActor0[0x91] == 2` (target_flag == 2, interpreted as "dead/despawning").

```c
// xiactor.cpp:1676
sceVu0SubVector(&fStack_50,&this->pos,pActor0 + 0x10);
if (fStack_10 < fStack_48*fStack_48 + fStack_50*fStack_50 + fStack_4c*fStack_4c) uVar1 = 0;
else if (pXStack_20[0x91] == 2) uVar1 = 0;   // dead target rejected
else uVar1 = 1;
```

**Server relevance:** the client will not let a player *select* or *retain* a target beyond
50 yalms (3-axis straight-line, no LOS in this particular check). LSB enforces target/spell
range server-side; this confirms the client's own hard cutoff is 50 yalms and that distance
is true 3D (height counts). An emulator that allowed actions beyond 50 yalms would diverge
from what an unmodified client can even request.

## 2. 6-yalm interaction / trade range  (squared = 36.0)

- `src/main/tk/TkTrade.cpp:1835` and `:1843`: trade-proximity uses `<= 36.0` (= 6.0²), both
  via an inline squared-distance compare and via `XiActor::CheckTargetDistance(...,36.0)`.
- `src/main/actor/xiactor.cpp:535` `XiActor::SysMove`: `fVar8 = 36.0;` used as a per-frame
  actor-proximity working value (sort / contact bookkeeping).

**Server relevance:** 6 yalms is FFXI's canonical trade / "близко" interaction distance.
LSB's trade and NPC-interact handlers should reject beyond ~6 yalms to match client behavior.

## 3. 8-yalm player-collision ("bump") radius + GM no-clip + chocobo no-collision

`XiControlActor::CheckContactActor` (xicontrolactor.cpp:1574-1675):

- `xicontrolactor.cpp:1601-1602`: **GM level >= 3 disables collision entirely**
  (`bVar1 = GetGmLevel(); if (bVar1 < 3) { ...collision... }`). GMs walk through players.
- `xicontrolactor.cpp:1614`: `fVar4 = 64.0;` — candidate search radius is 64.0 squared = **8.0
  yalms**. Final overlap test subtracts both actors' collision radii (GetCollisionSize) at
  `:1650-1652`.
- `xicontrolactor.cpp:1626-1627`: actors whose equip slot 0 value is in `[0x32, 0x3b]` (50–59)
  are skipped — that range is the **chocobo/mount body**, so you don't collide with mounted /
  riding actors. (Field interpretation inferred from value range; the equip-slot accessor name
  `GetEquipNum` is authoritative.)
- `xicontrolactor.cpp:1624-1625`: dead/event actors (GameStatus 2 and 3) are skipped.
- `xicontrolactor.cpp:1655`: contact hold timer initialized to `0x1e` = **30** (frames; ~0.5 s
  at 60 fps) — once you bump someone you stay "in contact" for 30 frames.

`GetCollisionSize` (xicontrolactor.cpp:3506-3518) reads model attribute `0x2c`, takes the XZ
magnitude, and multiplies by `DAT_0054c244` (value in .rodata, not in source).

**Server relevance:** purely client-side movement physics — the server does not police
player-vs-player bumping. The GM-level-3 no-clip is interesting as a client capability gate.

## 4. Client-side MOVEMENT SPEED CAP = 30 units/frame  (speedhack ceiling)

`XiControlActor::StepControl` (xicontrolactor.cpp:582-...):

```c
// xicontrolactor.cpp:630-638
lVar6 = XiActor::GetGameStatus((XiActor*)this);
if (lVar6 == 5) {                       // GameStatus 5 (mounted/chocobo, inferred)
    fVar8 = (**...0xcc)();               // base speed
    fVar8 = fVar8 * 2.0;                 // chocobo doubles speed
}
if ((EnableSpeedLimit != '\0') && (30.0 < fVar8)) {  // HARD CAP
    fVar8 = 30.0;
}
fVar8 = fVar8 / 60.0;                    // per-frame from per-second-ish
```

- `EnableSpeedLimit` (xicontrolactor.cpp:635) is the only reference in the codebase — a global
  toggle for the cap.
- The cap value is **30.0** before the `/60.0` normalization.
- `GetGameStatus()==5` (chocobo/riding, inferred) applies a **2.0×** speed multiplier.

`XiActor::Init` related: `xiactor.cpp:281` `(*...0xd0)(debug_actor_move_speed * 4.0)` and
`:282` uses `DAT_0054c1f0 * debug_actor_move_speed` — debug speed scaling (4× hook), present
in retail.

**Server relevance:** the legitimate client clamps run speed to 30 (scaled) units; a server
movement-speed sanity check can use this as the upper bound for an unmodified client (with
the chocobo 2× exception). Movement that consistently exceeds the implied per-tick distance is
not reachable by the stock client when `EnableSpeedLimit` is on.

## 5. Action / spell / ability ID validation caps (client gates before send)

The text-command layer validates the requested action ID range **before** queueing the action
packet (`gcZoneSendQueSearch(0x1a)`):

- **Magic ID**: `0 <= id <= 0x1ff` (511). `commandcalc.cpp:3429`
  `if (((lVar4 == 0) || (lVar3 < 0)) || (0x1ff < lVar3))  param_3 = -param_3;` (reject).
- **Weaponskill ("TECH") ID**: `0 <= id <= 0x1ff` (511). `commandcalc.cpp:3503`.
- **Job Ability ID**: `0 <= id <= 0x2ff` (767). `commandcalc.cpp:3575`.
- **Item index (for /item)**: `1 <= idx <= 0x50` (80, 'P'). `commandcalc.cpp:3313`
  `if (((lVar2 == 0) || (cVar1 < '\x01')) || ('P' < cVar1)) param_3 = -param_3;`.

Also in `XiControlActor::UseMagic` / `UseProcMagic` the animation/effect ID is clamped to
`< 0x1ff` (511) with an error path that resets it to 0:
- `xicontrolactor.cpp:2181-2184`, `:2314-2317`, `:2387-2389` (`ErrPrintf` then set to 0).
- `xicontrolactor.cpp:2165` `UseProcMagic` requires combo-effect index `< 0x10` (16).

**Server relevance:** these are the ID spaces the client believes exist (magic ≤511, WS ≤511,
abilities ≤767). LSB IDs outside these ranges could not be triggered by the stock client's
text path. They double as the field widths in the outgoing packet (see Tier 2).

---

# TIER 2 — Action packet format & category enum (client → server)

All player combat/action commands are built into the **zone send queue slot type `0x1a`** via
`gcZoneSendQueSearch(0x1a)` and committed with `gcZoneSendQueSet(slot, 0x10)` (0x10 = 16-byte
payload). The canonical builder is:

`makeActionPacket(ActIndex, UniqueNo, ActionID, ActionBuf)` — commandcalc.cpp:1685-1716
(symbol `makeActionPacket__FUiUiUsUi`):

```
slot+4  = UniqueNo   (uint, target's UniqueNo / CHAR_ID)
slot+8  = ActIndex   (ushort, target's actor index)
slot+10 = ActionID   (ushort, the CATEGORY — see table)
slot+0xc= ActionBuf  (uint, the action argument: spell/WS/ability ID, or 0)
```

This is FFXI's client "action" request (the family that the server answers with the 0x28
action result). The **ActionID is the action category**. Observed values (all authoritative —
literal stores at `slot+10`):

| ActionID | Meaning (from the cmdf_ function it's emitted in)        | Source (commandcalc.cpp) |
|---------:|----------------------------------------------------------|--------------------------|
| 0        | NPC interaction / "talk"/trigger (servtalk2NPC)          | :3748                    |
| 2        | Engage / attack ON (lock target & start auto-attack)     | :2842, :2880             |
| 3        | Cast magic (arg = spell id, ≤0x1ff)                       | :3446                    |
| 4        | Disengage / attack OFF (uses own CHAR_ID)                | :2936, :2976             |
| 5        | /heal — rest / kneel (uses own CHAR_ID)                  | :3266                    |
| 6        | (cmdf_COM_BATTLEDEBUG path)                              | :10418                   |
| 7        | Weaponskill ("TECH", arg = WS id, ≤0x1ff)               | :3519                    |
| 9        | Job ability (arg = ability id, ≤0x2ff)                  | :3586                    |
| 0xc (12) | /assist (target's target)                               | :3064                    |
| 0xe (14) | /fish (FISH)                                            | :3207                    |
| 0xf (15) | Switch attack target while already engaged              | :2908, :3025             |
| 0x10(16) | Ranged attack (/shoot, /ra)                             | :2751                    |
| 0x11(17) | /dig (chocobo digging)                                  | :3149                    |
| 0x12(18) | /dismount (DISMOUNT)                                     | :3103                    |
| 10 (0xa) | (cmdf_COM_JOB path)                                      | :10466                   |

(The grep of all `slot+10 = <literal>` stores: commandcalc.cpp lines 2751, 2842, 2880, 2908,
2936, 2976, 3025, 3064, 3103, 3149, 3207, 3266, 3446, 3519, 3586, 3748, 10418, 10466.)

These map onto FFXI's well-known action categories (engage=2, weaponskill, magic cast=3,
ability=9, ranged, etc.). I'm confident about engage/disengage/magic/WS/ability/ranged from
the function names; categories 6 and 10 come from debug/job commands and I'm **less certain**
of their exact server meaning.

**Server relevance:** this is the exact category numbering and 16-byte layout LSB must accept
for player action requests. Note the engage/disengage/target-switch split (2 / 4 / 0xf) and
that disengage and /heal send the player's OWN CHAR_ID as the "target."

## Client-side pre-send gating in the action commands

Before queuing, every combat command calls `CheckGameStatus__Fi(0x3f)` (a bitmask of allowed
game states) and several call `IsGameStatus(1)` (must be engaged/in-battle) or check
`IsEnemy__FP10XiAtelBuff`:

- `commandcalc.cpp:3408` (MAGIC), `:3483-3486` (TECH requires `IsGameStatus(1)` else AddErrMsg
  0xf "you must be engaged"), `:3555` (ABILITY), `:3293` (ITEM).
- `commandcalc.cpp:2862-2865` (engage): target must pass `IsEnemy`; else error 0x94.
- `commandcalc.cpp:2867` / `:2924`: branches on `*(field_0x124)==1` — the actor's current
  battle/animation state — choosing category 2 (engage) vs 0xf (switch) vs 0x15 path.

**Server relevance:** the stock client refuses to send a weaponskill unless it thinks the
player is engaged, refuses to engage a non-enemy, and gates everything on game state. A server
must still enforce these independently (a modified client can bypass the local checks).

---

# TIER 2b — ACTION RESULT packet (server → client, "0x28") bit-field layout

The action *result* is decoded by `CXiSchStatus::Unpack` (`Unpack__12CXiSchStatusFPv`,
**src/common/XiSchStatus.cpp:340-945**), reached from `RecvBattleCalc2`
(`RecvBattleCalc2__FP7GC_ZONEP19GP_GAME_PACKET_HEADP15GP_SERV_BATTLE2`,
**src/main/actor/xiatelnet.cpp:11162**, handling `GP_SERV_BATTLE2`). The packet is a custom
**LSB-bit-stream** read LSB-first one bit at a time (the inner `for` loops at
XiSchStatus.cpp:456ff shift each byte right by 1). I read out the exact bit widths from the
loops — these are **authoritative** (literal loop bounds) even though the field *names* below
are inferred from FFXI knowledge.

Top-level header (XiSchStatus.cpp:456-523):

| Field                | Bits | Stored at        | Meaning (inferred)                         |
|----------------------|-----:|------------------|--------------------------------------------|
| actor UniqueNo       |  32  | MainToCalc+0x04  | the acting entity's server ID              |
| target count         |   6  | +0x14            | number of targets (so **max 63** targets)  |
| category             |   4  | +0x10            | action category (**0–15**)                 |
| ? (param a)          |   4  | +0x08            | 4-bit field                                |
| param                |  32  | +0x0c            | command param (animation/spell, full 32b)  |

Per **target** (loop ×count, stride 0x14c) (XiSchStatus.cpp:526-560):

| Field             | Bits | Meaning                                            |
|-------------------|-----:|----------------------------------------------------|
| target UniqueNo   |  32  | target server ID (then resolved to actor pointer)  |
| result count      |   4  | sub-results for this target (**max 15**)           |

Per **result** (loop ×result-count, stride 0x28) (XiSchStatus.cpp:561-785) — **exact widths**:

| Field        | Bits | Off   | Meaning (inferred)                                            |
|--------------|-----:|-------|--------------------------------------------------------------|
| reaction     |   2  | +0x24 | hit/miss/evade reaction class                                |
| animation    |   2  | +0x28 | 2-bit                                                        |
| effect/anim  |  10  | +0x2a | 10-bit animation/effect id                                  |
| info         |   4  | +0x2c | 4-bit                                                        |
| **scale**    |   5  | +0x2e | knockback/scale index (**0–31**; cf. SetBlowBack 0–7)       |
| **value**    |  16  | +0x30 | **damage / heal amount — 16 bits → hard max 65535**          |
| message      |   9  | +0x34 | battle-log message id (**0–511**)                           |
| bit flags    |  32  | +0x38 | status/effect bit flags                                     |
| has-proc     |   1  | —     | if 0, the 4 proc fields are zeroed                          |
| proc_kind    |   6  | +0x3c | additional-effect type (only if has-proc)                  |
| proc_info    |   4  | +0x3e |                                                            |
| proc_value   |  14  | +0x40 | additional-effect value (**0–16383**)                      |
| proc_message |   9  | +0x42 | additional-effect message id                               |
| has-react    |   1  | —     | if 0, the 4 react fields are zeroed                         |
| react_kind   |   6  | +0x44 | counter/spikes reaction type                               |
| react_info   |   4  | +0x46 |                                                            |
| react_value  |  14  | +0x48 | reaction value (**0–16383**)                               |
| react_message|   9  | +0x4a | reaction message id                                        |

**Server relevance (significant):**
- **Damage/heal `value` is 16-bit → the client cannot display a hit larger than 65535.** Any
  server damage value is effectively truncated to `value & 0xFFFF` on the wire. The classic
  FFXI "additional effect" and "spikes/counter" damages are the separate **14-bit**
  `proc_value`/`react_value` (max 16383). An emulator producing larger numbers will wrap.
- **Target count is 6 bits (≤63)** and **per-target results 4 bits (≤15)** — packet structural
  caps the client expects.
- **category is only 4 bits (0–15)** in the *result* packet — narrower than the request side
  (which used values up to 0x12). The decode of `m_uCmdNo` (the result-side category) is the
  switch in **RecvBattleCalc2, xiatelnet.cpp:11224-11285** (verified directly):

  | m_uCmdNo                | Handler vtable slot called          | Meaning (inferred)                    |
  |-------------------------|-------------------------------------|---------------------------------------|
  | 1, 2                    | `*(this+0x178)` (attack result)     | **melee / auto-attack** round         |
  | 3, 4, 5, 6, 0xb, 0xd    | `*(this+0x174)` (magic/effect)      | **magic & effect** family             |
  | 7, 8, 9, 10 (0xa)       | `*(this+0x154)` (SetAction anim)    | **start** of cast/ability/item/WS     |
  | 0xc (12)                | `*(this+0x154)`                     | (special, also SetAction)             |
  | default                 | discard packet                      | unknown category → dropped            |

  In the 7/8/9/10 branch (xiatelnet.cpp:11254-11277) the client treats
  `m_uCmdArg & 0xffff == 0x6163` (ASCII `"ca"`) as the **"action accepted / begin"** marker:
  on that value it calls the *use* callbacks (`KaMenuMagicUseCallBack` for cmd 8,
  `KaMenuAbilityUseCallBack` for cmd 10) and `SetCastMagicID`; otherwise it fires the matching
  **cancel** callbacks (`...CancelCallBack`, `YkWndItemUseCancelCallBack` for item cmd 9).
  So `m_uCmdNo` 8 = magic-cast start, 9 = item-use start, 10 = ability start, and the
  `0x6163` sentinel in the arg distinguishes start vs interrupt. The `value` read here
  (`m_stResult.value & 0xffff`) carries the spell/ability id for the menu recast UI.
- The two optional blocks (proc / react) each gated by a **1-bit present flag** — matches
  FFXI's "additional effect" and "spikes" optional sub-messages. LSB must set these flags
  correctly or the client will misalign the bit stream for subsequent fields/targets.

---

# TIER 3 — Targeting model & cursor cycling

## Target search uses inverse view-matrix screen space

`GetTargetAuto`, `GetTargetScreen`, `GetTargetNearestPC/NPC` all build the control actor's
inverse view matrix (`sceVu0InversMatrix`) and require the candidate to be **in front**
(`0.0 <= transformed.x`, e.g. xiactor.cpp:1095, 1498, 1598) and to have non-negative `depth`
(on-screen). Candidate filtering flags (the `Type`/`Pos` bitmask argument):

- bit `0x1` → exclude self / a "no-PC?" class (paired with `field_0xfe` sign test)
- bit `0x10` → exclude enemies (`IsEnemy__FP10XiAtelBuff`)
- bit `0x200` (GetTargetScreen only, :1176) → allow self as target
- monster actors in GameStatus 2 or 3 (dead/despawning) are excluded everywhere.

`GetTargetCursol` (xiactor.cpp:1630-1645): an actor is "uncursorable" if it has no atel buffer,
is a sub-actor, or a specific bit of `atel_work->field_0xf8` is set (the not-targetable flag).

## Screen-center constant 2048 and fixed-point >>4 coordinates

`GetTargetScreen` (xiactor.cpp:1154-1159) maps to a virtual screen using `... * 0.5 + 2048.0`
on each axis — **2048 is the screen-center origin** in the client's projected coordinate space.

`XiControlActor::Targetting` (xicontrolactor.cpp:2568-2607) — the auto-target-nearest routine:
```c
uVar3 = 0x400000;                 // initial "max distance" (4194304) for nearest search
...
iStack_30 = iStack_30 >> 4;       // positions are 1/16 fixed-point (divide by 16)
iStack_2c = iStack_2c >> 4;
fVar4 = sqrtf((iStack_30-0x800)^2 + (iStack_2c-0x800)^2);  // 0x800 = 2048 center
```
So the on-screen targeting grid centers on **0x800 (2048)** and coordinates are **>>4
(1/16-unit fixed point)**.

## Left/Right target cycling scoring

`GetTargetLeft` / `GetTargetRight` (xiactor.cpp:1200-1419): cycling left/right among targets.
- `fStack_40 = 3600.0;` (xiactor.cpp:1258, 1370) — angular-score init.
- candidate distance gate `< 2500.0` (50 yalms²) as above.
- a minimum-divisor clamp of `1.0` to avoid divide-by-zero on near-zero lateral offset
  (xiactor.cpp:1253-1255, 1289-1291, etc.).
- horizontal screen weight `fRam00557a28` (value in .rodata).

**Server relevance:** target cycling and "nearest" selection are 100% client-side; the server
only sees the resulting CHAR_ID. Useful context for emulator devs reasoning about which target
a client would plausibly pick, but not a rule the server enforces.

---

# TIER 4 — Melee reach, approach-to-target, knockback

## Attack reach is data-driven (model attribute 0x19)

`XiSkeletonActor::GetAttackReach` (xisklactor.cpp:7811-7823) reads model attribute index
`0x19`, takes the vector magnitude (`sceVu0InnerProduct` + `KO_Sqrt`). So **melee reach is
per-model**, not a single global constant — it comes from the skeleton/model resource.

It's consumed by `XiControlActor::ApproachControl` (xicontrolactor.cpp:2687, 2714): the
auto-approach when engaging walks the player toward the target until
`distance - GetAttackReach() <= 0`, i.e. stops exactly at attack reach. `DAT_0054c1fc` is a
small "snap" epsilon used as the close-enough band (xicontrolactor.cpp:2716; value in .rodata).

`Approach`/`BackJump` (xicontrolactor.cpp:2841-2875) set `approach_target/timer/speed`. The
per-frame mover uses `approach_speed * iRam00557a44` (frame delta) and clamps the final step to
not overshoot (xicontrolactor.cpp:2723-2727).

**Server relevance:** there is no single hardcoded melee range in the client — it's the model's
reach attribute. LSB uses a model-size-based reach too; this confirms the client also derives
reach from the model rather than a flat number, so an emulator's per-mob reach should track
model size to match client auto-approach behavior.

## Knockback ("blow back") — 8-entry parameter table, state gating, no-knockback states

`XiControlActor::SetBlowBack(actor, resid, is_rev, scale_no)` (xicontrolactor.cpp:3160-3176):
- `scale_no` indexes **`BlowBackParaTab`**, stride **0xc (12) bytes**, **8 entries**
  (`if (scale_no < 8)`), each entry = `{power(float), dumper(float), timer(float)}`
  (xicontrolactor.cpp:3166-3173). Values live in .rodata (not in source). `is_rev` negates the
  power (pull instead of push).

`SetBlowBack(...,power,dumper,timer)` (xicontrolactor.cpp:3191-3236):
- Knockback is **suppressed** when the victim's GameStatus is one of
  `0x2b,0x2a,0x29,0x28,0x27,0x26,6,5` (xicontrolactor.cpp:3202-3205) — i.e. dead / event /
  special action states (the 0x26-0x2b cluster) and status 5/6.
- timer 0 is coerced to `1.0` (xicontrolactor.cpp:3223-3224).
- while active the actor is `AddLockStatus`'d (action-locked).

`BlowBackControl` (xicontrolactor.cpp:3251-3387): integrates the knockback each frame, scaling
the velocity by `blowback_dumper` (damping) per frame and ending when speed² drops below
`DAT_0054c204` (value in .rodata). For "pull" (power<0) it stops at distance² `DAT_0054c200`.

**Server relevance:** knockback magnitude is a client-side table (8 strengths) keyed off the
action resource — the server's knockback "scale" parameter selects one of these 8 entries.
Worth knowing the index space is 0–7 and that knockback is animation-locking on the client.

---

# TIER 5 — Action animation lock & "moving" actions

- `XiControlActor::IsMovingTechnic` (xicontrolactor.cpp:2093-2117): true if a scheduler task
  whose finisher is `FinishTechnic` is active — i.e. a weaponskill/technic animation is
  running. Used to know the player is mid-WS.
- `XiActor::IsMovingAction` (xiactor.cpp:2358) and `SetAction`/`KillAction`
  (xiactor.cpp:2293-2356) manage generic action animations.
- `XiActor::AddLockStatus` / `SubLockStatus` (xiactor.cpp:753-822) is the action lock; magic
  casting (`UseMagic`/`UseProcMagic`) and knockback both `AddLockStatus` the caster and all
  targets while the resource loads / effect plays.
- `IsControlLock` (xicontrolactor.cpp:3130-3145): a **counted** (nesting) control-lock — input
  is disabled while `is_control_lock > 0`.

`UseMagic` (xicontrolactor.cpp:2238-2448) maps action sub-type → animation resource base:
- type 3 → `0xd48` (skeleton technic/WS path; uses `ReadTechRes`)
- type 4/8 → `0xaf0`
- type 5/9 → `0x1330`
- type 6/10 → `0x113c`
- type 7/11 → `0xf3c` (with extended table `0xbe57` when id > 0x1ff)
- type 0xd → `0xbd57`
(xicontrolactor.cpp:2265-2290.) These are resource IDs, not gameplay numbers, but they show
how the client routes each action category to an animation set and that IDs >511 fall into an
"extended" resource block.

**Server relevance:** mostly client animation timing. The action lock means the client won't
fire a second action mid-animation, but the **server must still enforce its own recast /
action timers** — the client lock is cosmetic/UX, not authoritative.

---

# TIER 6 — Client-side action permission gates (`CanIMove` / `CanITarget`)

`CanIMove` (xiatelnet.cpp:7824-7942) — whether the player may currently move (and is the gate
`StepControl` honors via `CanIMove()` at xicontrolactor.cpp:652, and `SetTarget` via the
related `CanITarget`):

Movement is **blocked** when any of these hold (notable subset):
- not logged in / no actor (`_LoginActIndex == 0`).
- `field_0xc2` (current animation/pose state, inferred) is 3, 4, or 5; OR `field_0x158`
  (action state) is one of `0x14,0x10,0x0f,0x08,0x07,0x06,0x05,0x04,0x02,0x01`
  (xiatelnet.cpp:7841-7845) — i.e. mid-action / special poses.
- an active client event (`CliEventUcFlag`, `CliEventUcFlag2`, `EventExecFlag`).
- a movement-disabling **buff status** is set — checked via `KaComCheckBuffStatus(id)` for ids
  `0, 2, 7, 0xa(10), 0x1c(28), 0xe(14)` (xiatelnet.cpp:7857-7871). In FFXI these are the
  "can't act/move" statuses; I'm **confident 2 = Sleep** and these are the stun/bind/petrify
  family, but I have **not** verified each id against the LSB status enum, so treat the exact
  mapping as inferred.
- current/queued action state `field_0x124`/`field_0x120` ∈ `{0x2b,0x2a,0x29,0x28,0x27,0x26,
  6,0x2c}` (the same dead/event cluster as knockback) (xiatelnet.cpp:7874-7885).
- mailbox/delivery (`CTkDelivery::IsOpenDelivery`, `CTkPost::IsOpenPost`) or a specific
  not-movable flag bit (`field_0xf9`/`field_0xfb`).

`CanITarget` (xiatelnet.cpp:7955-7959) **just `return 1;`** — targeting is never blocked at
this layer (targeting restrictions live in the per-candidate filters of Tier 3 instead).

**Server relevance:** the client voluntarily stops sending movement under Sleep/Bind/Stun/etc.
and during actions/events. The server MUST enforce these movement locks itself — a hacked
client that ignores `CanIMove` could move while bound/stunned, so LSB's bind/stun/sleep
handling cannot rely on the client honoring this.

---

# Items I looked for but did NOT find (scope notes / honesty)

- **The 8-yalm *vertical* AOE height limit** (the example finding given in the brief): I did
  **not** find it in `commandcalc.cpp`, `xiactor.cpp/.h`, `xicontrolactor.cpp`, or
  `xisklactor.cpp`. The AOE target-set selection (who is inside a spell/WS radius) appears to
  be computed **server-side** in FFXI — the client only sends the primary target CHAR_ID
  (ActionBuf carries the spell/WS id, not an area). The closest spatial limits the client
  enforces are the **50-yalm** target validity (Tier 1.1) and the **8-yalm** *collision*
  radius (Tier 1.3), neither of which is the AOE height clamp. If the 8-yalm height rule
  exists in this client it is likely in the AOE effect-application code (effect/scheduler
  resources) or simply not present client-side. Worth a follow-up grep of `xievent.cpp` /
  effect scheduler files.
- **Damage display caps / hit-vs-miss masks / DAMAGE_KIND enum values**: the actor files
  themselves only consume an already-decoded result to play animations (`SetAttack`
  xisklactor.cpp:4816/4893 just plays the swing and prints the battle message string). The
  actual decode lives in `CXiSchStatus::Unpack` (src/common/XiSchStatus.cpp) and
  `RecvBattleCalc2` (xiatelnet.cpp:11162) — now documented in **Tier 2b** above. The damage
  cap **is** real: the result `value` field is **16 bits (max 65535)**. I did not find a named
  C `enum DAMAGE_KIND` with explicit values in source; `DAMAGE_KIND` appears only as a
  parameter *type* in mangled signatures (e.g. `SetAttack__...11DAMAGE_KIND`), so its enum
  constants are not recoverable from these files.
- **Recast/cast-time constants**: not in the actor files. `/recast` reads them via
  `KaMenuAbilityGetRecastTime` / `KaMenuMagicGetRecastTime` (commandcalc.cpp:3617-3619), i.e.
  the akaza menu subsystem owns recast data.

# .rodata constants referenced but whose values are not in the decomp source

These addresses are used as floats/tables in the combat code; the maintainer may want to dump
them from the binary to recover exact values:

- `DAT_0054c1fc` — approach/close-enough epsilon (ApproachControl, BackJumpControl, StepControl)
- `DAT_0054c1f4`, `DAT_0054c1f8` — direction/threshold bounds (GetApproachPoint, StepControl)
- `DAT_0054c200`, `DAT_0054c204` — knockback distance²/speed² end thresholds (BlowBackControl)
- `DAT_0054c208` — turn-rate factor (StepControl)
- `DAT_0054c244` — collision-size multiplier (GetCollisionSize)
- `DAT_0054c1f0` — actor move-speed scale (XiActor init)
- `BlowBackParaTab` — 8 × {power,dumper,timer} float knockback table
- `GA_WAZA_COMBO_EFFECT_3339` — skillchain/combo effect lookup (UseProcMagic, ≤16 entries)
- `MagicCateTbl`, `BlowBackParaTab`, `fRam00557a28` (screen H-weight)
