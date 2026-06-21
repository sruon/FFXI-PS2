# FFXI PS2 Client (SCUS_972.66) — Effects, Events & Enums

Scope: status-effect / animation / sound scheduling; the EVENT system (NPC
interaction, cutscenes, menus, the event protocol); and the global enum / type /
constant tables. Target audience: maintainers of a LandSandBoat (LSB) server
emulator who need the **client-side meaning** of effect IDs, event opcodes, and
the action packet.

**Provenance key**
- *DWARF-authoritative*: name comes from original PS2 DWARF1 debug symbols
  (function names, struct field names in `BattleResult`, `CXiMainToCalc`, etc.).
- *Inferred*: derived from reading the decompiled switch/case logic and address
  constants; values are exact (read from code) but the *label* is my
  interpretation.
- Ghidra did **not** recover C `enum` enumerator lists. The "127 enums in
  _types.h" premise does not hold for this tree — there is no `include/_types.h`;
  enums are scattered as forward-declared names across `include/*.h` with empty
  bodies (e.g. `enum GAME_STATUS`, `enum DAMAGE_KIND`, `enum RESULT_MES_TYPE`).
  The real values live in the `.cpp` switch statements catalogued below.

---

## 1. EVENT PROTOCOL (highest LSB relevance)

The client event system is a **bytecode virtual machine** (`XiEvent`) driven by
a per-event compiled script loaded from disk, started by a server packet, and
terminated by sending a result packet back. This is exactly the layer LSB's
event scripting (`onTrigger` / `updateEvent` / `eventFinish`) must mirror.

### 1.1 Receiving an event (server → client)

Three inbound packet handlers, all in `src/main/actor/xiatelnet.cpp`. They differ
only in the extra payload they carry; all set the same four globals and call
`InitEvent`.

| Packet struct | Handler (file:line) | Extra payload |
|---|---|---|
| `GP_SERV_EVENT`    | `RecvEventCalc` — xiatelnet.cpp:12276 | none (plain event) |
| `GP_SERV_EVENTNUM` | `RecvEventCalcNum` — xiatelnet.cpp:12373 | 8 × u32 "params" → `Work_Zone[2..9]` |
| `GP_SERV_EVENTSTR` | `RecvEventCalcStr` — xiatelnet.cpp:12419 | 4 × 16-byte strings → `EventStrBuf[0..3]` |

Common fields read by all three (offsets are into the packet body; field names
*inferred* from globals they feed):

```
RecvEventCalc (xiatelnet.cpp:12291-12296):
  _CliEventMode      = u16 @ +0x0e   // behaviour bit-flags, see 1.5
  _CliEventNum       = u16 @ +0x0a   // event ID  -> InitEvent arg1
  _CliEventPara      = u16 @ +0x0c   // event param/"mode" -> InitEvent arg2
  _CliEventUniqueNo  = u32 @ +0x04   // target NPC server UniqueNo
  _CliEventIndex     = u16 @ +0x08   // target NPC local actor index
  InitEvent(_CliEventNum, _CliEventPara)
```

`RecvEventCalcNum` uses different offsets (+0x2a num, +0x2c para, +0x04 unique,
+0x28 index, +0x2e mode; params at +0x08..+0x24). All three gate on
`EventExecFlag == 0` and `_app == 0x60` (the "in zone / game" app phase); a busy
client replies with a "you cannot do that now" system message (xiatelnet.cpp:12304).

**Server relevance:** `_CliEventNum`/`_CliEventPara` are LSB's `csid` and
`menuid`/param. The "Num" variant is the standard *event-with-params* packet
(LSB `startEvent(id, ...params)`); the "Str" variant is *event-with-strings*
(LSB `startEventString`). UniqueNo is the entity the event "belongs to".

### 1.2 Loading the script & messages

`InitEvent__FUsUs(eventnum, para)` — xievent.cpp:2416. Resets ~40 event flags,
clears `Work_Zone[0x10..0x60]`, then loads two files by ID:

```
xievent.cpp:2457  event bytecode  = file id (eventnum + 0x16bc)
xievent.cpp:2464  event messages  = file id (eventnum + 0x1914)
```

So **event script #N is DAT file 0x16bc+N** and its **dialogue table is
0x1914+N**. (Inferred file-ID bases; exact constants from code.)

### 1.3 The event bytecode VM — opcode table

`XiEvent::ExecProg(this, opcode)` — xievent.cpp:6646. Giant `switch(param_2)`
where `param_2` is the opcode byte fetched from `EventData[ExecPointer]`. The VM
has: local work registers `Work_Local[0..0x800]`, zone work `Work_Zone`
(0x1000-0x1800 window), special slots `0x7f00+` for event position/direction
(setworkofs/getworkofs, xievent.cpp:2669 / 2717), an 8-deep gosub stack
(`GosubRetAdrs`, `GosubStackPtr`), and per-request state in `Req[RunPos]`.

Opcodes (case value → meaning; all *inferred* from code, exact case numbers):

| Op | Meaning | Op | Meaning |
|----|---------|----|---------|
| 0x00 | END request / RetFlag=1 | 0x01 | JMP (abs addr u16) |
| 0x02 | IF (CodeIF, xievent.cpp:3058) | 0x03 | MOV work=imm |
| 0x04 | NOP(+3) | 0x05/0x06 | SET work=1 / =0 |
| 0x07 | ADD | 0x08 | SUB |
| 0x09 | BITSET (1<<n) | 0x0a | BITCLR |
| 0x0b | INC | 0x0c | DEC |
| 0x0d | AND | 0x0e | OR | 0x0f | XOR |
| 0x10 | SHL | 0x11 | SHR |
| 0x12 | RAND (rand()) | 0x13 | RAND mod (n+1) |
| 0x14 | MUL | 0x15 | DIV |
| 0x16 | SIN | 0x17 | COS | 0x18 | ATAN2 |
| 0x19 | SWAP two works | 0x1a | GOSUB (push) |
| 0x1b | RETURN (pop; if empty, end) | 0x1c | WAIT n frames |
| 0x1d | print/open event message (actor) | 0x1e | actor look-at + set dir |
| 0x1f | MOVE (CodeMOVE) | 0x20 | set CliEventUcFlag |
| 0x21 | EventExecEnd=1 | 0x22 | actor hide flag |
| 0x23 | MESWAIT | **0x24 | QUERY — open menu (CodeQUERY, xievent.cpp:3350)** |
| 0x25 | QUERYWAIT (CodeQUERYWAIT) | 0x26 | yield (RetFlag=1) |
| 0x27 | REQ set on actor (REQSet) | 0x28/0x29 | REQSW / REQEW |
| 0x2a | check REQ level | 0x2b | open message on actor |
| 0x2c | SCHEDULOR (play effect schedule) | 0x2d | MAPSCHEDULOR |
| 0x2e | set cancel flag | 0x2f | actor flag bit |
| 0x30 | clear ucoff_continue | 0x31 | SMOVE |
| 0x32 | set MainSpeed = n/10 | 0x33 | actor flag |
| 0x34 | **LOAD ZONE/MAP** (delete all actors, XiZone::Open) | 0x35 | same (variant) |
| 0x36 | set event pos (x,z,y) | 0x37 | set event pos + dir |
| 0x38 | set _CliEventModeLocal | 0x39 | set event dir |
| 0x3a | get actor dir → work | 0x3b | get actor pos → work |
| 0x3c | array BITSET | 0x3d | array BITCLR | 0x3e | array BITTEST branch |
| 0x3f | MOD | 0x40 | SETBITWORK | 0x41 | GETBITWORK |
| 0x42 | clear cancel | 0x43 | **pending-tag send/wait** (SendPendingTag) |
| 0x44 | branch if actor not found | 0x45 | LOADSCHEDULER |
| 0x46 | DEFCAMERA | 0x47 | **pending XYZ send** (SendPendingXzyTag) |
| 0x48 | open message (no actor) | 0x49 | open message on actor (work id) |
| 0x4a | DTURA | 0x4b | set actor dir from work |
| 0x4c/0x4d | set actor game_status = 8 / 9 (door open/close) | 0x4e | actor hide |
| 0x4f | set actor status = n+0x12 (ship/transport) | 0x50 | ENDSCHEDULOR |
| 0x51 | ENDMAPSCHEDULOR | 0x52 | ENDLOADSCHEDULER |
| 0x53 | WAITSCHEDULOR | 0x54 | WAITMAPSCHEDULOR | 0x55 | WAITLOADSCHEDULER |
| 0x56 | resolve actor index (yield) | 0x57 | add frame-time to work |
| 0x58 | yield 1 frame | 0x59 | set actor turn speed |
| 0x5a | MOVE2 | **0x5b | EXTSCHEDULER load** |
| 0x5c | set MusicBuff slot | 0x5d | music volume |
| 0x5e | set actor LastAction / def motion | 0x5f/0x60 | actor flag bits |
| 0x61 | actor flag | 0x62 | LOADEVENTSCHEDULER |
| 0x63 | actor emote/face (status 9) | 0x64 | distance(x1,y1,x2,y2) |
| 0x65 | GETDISTANCEAA | 0x66 | EXTSCHEDULER(1) |
| 0x67 | open compass/preset event message mode | 0x68 | close it |
| 0x69 | sound volume on/off (effect/system/zone/master bits) | 0x6a | sound volume fade |
| 0x6b | set actor motion (work id) | 0x6c | TRANSPAR (alpha) |
| 0x6d | NOP(+7) | 0x6e | EMOT (emote) |
| 0x6f | WAIT ~16 frames | 0x70 | TurnCancel (self) |
| 0x71 | OPENPASSWIN (password window) | 0x72 | GETWEATER (weather→work) |
| 0x73 | MAGICSCHEDULOR | 0x74 | actor flag |
| 0x75 | LOADROOM (Mog House room) | 0x76 | TurnCancel (actor) |
| 0x77 | **set game time / weather** (CodeENVIRONMENT) | 0x78 | reset time/weather |
| 0x79 | look-at (two actor ids) | 0x7a | look-at with speed |
| 0x7b | clear actor look | 0x7c | actor flag (toggle) |
| 0x7d | LoadStartScheduler 'main' | 0x7e | CHOCOBO |
| 0x7f | QUERYWAIT2 | 0x80 | LOADWAIT |
| 0x81 | actor sub-flag | 0x82 | range-rect hit test branch (TsFindRangeRect) |
| 0x83 | get game time → work (ntGameTimeGet) | 0x84 | actor flag |
| 0x85 | **OpenMyroomMenu** (Mog House menu) | 0x86 | actor flag |
| 0x87 | **friend-pass send/wait** (gcZoneSendQue 0x1b) | 0x88 | (continues) |

(Opcodes ≥0x88 continue past xievent.cpp:8089 — same pattern: actor flags,
pending sends, more scheduler variants. The map above covers the gameplay-
relevant set.)

**Server relevance:** This is the canonical client interpreter for *compiled*
event scripts that ship in the client DAT files. LSB does NOT send this
bytecode — the bytecode is already on the client; LSB only sends the
*start* packet (event id + params) and receives the *result*. The opcode table
matters for understanding what a given event id will *do* on the client
(open a menu via 0x24, change zone via 0x34, play a magic schedule via 0x73,
gate on a server "pending" reply via 0x43/0x47/0x87). The pending opcodes
(0x43, 0x47, 0x87) are where the script blocks waiting for an LSB
`updateEvent`/extra packet.

### 1.4 The menu/query (CodeQUERY) and result send-back

`XiEvent::CodeQUERY` — xievent.cpp:3350. Two-phase:
1. First call: sends a query-request system packet (`TK_DATAHEDDER 0x99`),
   sets `QueryReqFlag`, yields.
2. When `QueryEndFlag` set: builds the menu window
   (`CTkMenuMng::CreateQueryWindow`). It walks `QueryBuf` and, for each
   option, calls `CTkQueryControl::AddItem` (selectable line) or `AddComment`
   (non-selectable prompt text). A bitmask from `getworkofs(this,5,0)`
   disables individual options (`uVar4 & 1` per option, shifted). The default
   cursor row comes from `getworkofs(this,3,0)`.

The selected option is returned to the server by **`SendEventEnd`** —
xiatelnet.cpp:12572. Outbound packet via `gcZoneSendQueSearch(0x5b, ...)`,
length 0x14:

```
  +0x04  u32  _CliEventUniqueNo   (target entity)
  +0x08  u32  result/option value  (uRam00673e04; = 0x40000000 if CancelEvent)
  +0x0c  u16  _CliEventIndex
  +0x0e  u16  0   (1 for SendPendingTag, see below)
  +0x10  u16  _CliEventNum   (event id)
  +0x12  u16  _CliEventPara  (event param)
```

Related senders (same packet 0x5b / 0x5c family):
- `SendPendingTag` — xiatelnet.cpp:12464: same layout, `+0x0e = 1` (mid-event
  "continue, give me more data" — LSB responds with another event packet).
- `SendPendingXzyTag` — xiatelnet.cpp:12532: packet 0x5c, length 0x20, adds
  x/y/z floats (+4/+8/+0xc) and direction byte (+0x1f). Used by opcode 0x47
  for position-pick events (e.g. furniture placement, conquest).

**Server relevance:** Packet **0x5b is the client→server "event update /
finish"** carrying the option index in the +0x08 result field. This is what
LSB receives as the player's menu choice (the value LSB passes to
`onEventUpdate` / `onEventFinish`). A cancel is signalled by result =
`0x40000000`. The +0x0e flag distinguishes *finish* (0) from *update/pending*
(1).

### 1.5 `_CliEventMode` behaviour flags (inferred from usage)

`_CliEventMode | _CliEventModeLocal` is a bitmask gating client behaviour
during the event. Observed bits:
- `0x40`  — gate in several Recv* handlers (xiatelnet.cpp:12784/12862/12938/...);
  controls whether incoming packets are processed during the event.
- `0x100` — in event char setup (xievent.cpp:2600) and getworkofs path: skip
  certain actor init.
- `0x400` — RecvBattleCalc2 (xiatelnet.cpp:11221): suppress action effects on
  hidden actors during event.
- `0x1000`— RecvBattleCalc2 (xiatelnet.cpp:11236): allow self-action animation
  during event.

**Server relevance:** these are the "event flags" LSB passes in the start packet
(the `EventMode`/flags word). They control whether the player can move, whether
the world keeps updating, etc.

### 1.6 Event-character request (packet 0x16)

During event setup the client requests model/data for any NPC referenced by the
script that isn't loaded, via `gcZoneSendQueSearch(0x16, index, index)`
(xievent.cpp:2577, 2363), length 8, writing the actor index at +2. Guarded by
`EventCharReqFlag` bitset + `_EventCharReqCounter = 0x78` throttle.

**Server relevance:** Packet 0x16 = client asking the server to spawn/send a
specific NPC entity needed for the cutscene.

---

## 2. THE ACTION / BATTLE PACKET (status-effect, animation, damage)

This is the most important table for combat. The server's action packet
(GP_SERV_BATTLE2, the FFXI "0x28 action packet") is parsed by
`CXiSchStatus::Unpack` — `src/common/XiSchStatus.cpp:351`. It is a **LSB-first
bit-packed** stream (each field read bit-by-bit, LSB first; the unpacker shifts
the source byte right and advances every 8 bits — XiSchStatus.cpp:456-464).

Entry: `RecvBattleCalc2` — xiatelnet.cpp:11171 → `Unpack(packet+4)`.

### 2.1 Bit layout (exact widths from XiSchStatus.cpp; field names from BattleResult.h / CXiMainToCalc.h = DWARF-authoritative)

Header:

| Bits | Field | Dest off | XiSchStatus.cpp |
|------|-------|----------|-----------------|
| 32 | **actor / caster server ID** (`m_uID`) | +0x04 | :456 |
| 6  | **target count** (`m_sTargetSum`) | +0x14 | :479 |
| 4  | **category** = `m_uCmdNo` (action category) | +0x08 | :490 |
| 4  | `m_uInfo` | +0x10 | :501 |
| 32 | **param / "command arg"** = `m_uCmdArg` (spell/ability/WS id) | +0x0c | :514 |

Per target (loops `m_sTargetSum` times, stride 0x14c — XiSchStatus.cpp:526):

| Bits | Field | XiSchStatus.cpp |
|------|-------|-----------------|
| 32 | **target server ID** | :528 |
| 4  | **result count** | :551 |

Per result (loops result-count times, stride 0x28 — XiSchStatus.cpp:561):

| Bits | Field (BattleResult name) | XiSchStatus.cpp |
|------|---------------------------|-----------------|
| 2  | `miss` / reaction | :563 |
| 2  | `kind` | :574 |
| 10 | `info` | :586 |
| 4  | `scale` (sub_kind/scale) | :598 |
| 5  | (reaction extra) | :610 |
| 16 | `value` (damage / HP / MP / etc.) | :622 |
| 9  | `message` low | :633 |
| 32 | `bit` / animation+message extra | :644 |
| 1  | **has-additional-effect flag** | :655 |
| └ if set: 6 | `proc_kind` (additional-effect kind) | :672 |
| └ 4  | `proc_info` | :684 |
| └ 14 | `proc_value` | :696 |
| └ 9  | `proc_message` | :708 |
| 1  | **has-spike/react flag** | :721 |
| └ if set: 6 | `react_kind` | :738 |
| └ 4  | `react_info` | :750 |
| └ 14 | `react_value` | :762 |
| └ 9  | `react_message` | :774 |

These widths (32,6,4,4,32 / target 32 / per-result 2,2,10,4,5,16,9,32, then the
two flag-gated 6/4/14/9 blocks) **match the documented FFXI action packet exactly**
and are what LSB's action-packet builder must produce.

### 2.2 Category (`m_uCmdNo`) dispatch — client interpretation

`RecvBattleCalc2` switch — xiatelnet.cpp:11224 (values *inferred* from the
animation method each invokes):

| Category | Client action |
|----------|---------------|
| 1, 2 | melee attack round — vtbl+0x178 (`KillLastAction`/`SetAttack`) |
| 3,4,5,6,0xb,0xd | ongoing animation — vtbl+0x174 (replace attack/magic) |
| 7,8,9,10 | **action START** — vtbl+0x154 (`SetAction`), with sub-dispatch: cmdNo 8 = magic, 10 = ability, 9 = item; `m_uCmdArg & 0xffff == 0x6163` ("ac") distinguishes use vs cancel (xiatelnet.cpp:11254) |
| 0xc | ranged-style start — vtbl+0x154 |

Use/Cancel callbacks fired for the control actor (xiatelnet.cpp:11257-11275):
`KaMenuAbilityUseCallBack`, `KaMenuMagicUseCallBack`, `KaMenuAbilityCancelCallBack`,
`KaMenuMagicCancelCallBack`, `YkWndItemUseCancelCallBack`.

**Server relevance:** `m_uCmdNo` is LSB's action `category`; `m_uCmdArg` is the
action id (spell/WS/ability/item). Categories 7-10 are the "starts casting / readies"
animations; 1-6 are the result/finish animations.

### 2.3 Battle message packets

`RecvBattleMessage` — xiatelnet.cpp:11380; `RecvBattleMessage2` —
xiatelnet.cpp:11499. Floating combat text. `GetMessageColor__FUsUsi`
(xiatelnet.cpp:11302) picks a color via 7 tables (`MessageColorType` ..
`MessageColorType7`) based on caster/target relationship (self / party /
alliance / other) — indexed by message number.

**Server relevance:** message ids in the action packet (`message` field) index
the client's message DAT; LSB sends the numeric id, client renders the text +
color.

---

## 3. STATUS EFFECTS / BUFF ICONS

### 3.1 The status/buff icon array (packet)

`RecvCliStatus2` — xiatelnet.cpp:9282:
```
  64 × u16  status icons   -> 0x5c934a[0..0x40]   (packet +0x80, 2 bytes each)
  31 × u32  ability recast -> 0x5c93cc[0..0x1f]   (packet +0x04)
  then KaMenuAbilitySetRecastTime(0x5c93cc)
```
`RecvCliStatus` — xiatelnet.cpp:9312: the broader self-status packet (HP/MP/TP,
job levels at +0x14 (7×u16) and sub at +0x22, stats at +0x34 (8×u16), etc.;
fires `_CliStatusCallPtr` callback).

`SendBuffCancel__FUs(BuffNo)` — xiatelnet.cpp:12495: outbound packet 0x0f1,
length 8, BuffNo at +4; clears that buff from the local `CliStatus` array
(32 slots of u16, xiatelnet.cpp:12509). 0xffff = empty slot.

**Server relevance:** The status-icon array is **64 entries of u16 buff ids**
(LSB's status-effect icon list). Client-side these are pure icon ids; the buff
*meaning* is server-side. `CliStatus` (the player's own) is 32 slots. Packet
0x0f1 is the **client→server "cancel this buff"** (right-click status icon).

### 3.2 GAME_STATUS enum (entity status / motion state)

`XiAtelBuff::SetStatusMotion(GAME_STATUS)` — xiatelnet.cpp:1848. Switch on the
actor's `game_status` (field +0x124). Values (*inferred*; the motion it plays is
named by 4-char packed motion codes from string table
`s__initini1ini2ini3seq...` at 0x4d987f):

| game_status | meaning (client motion) |
|---|---|
| 0,4,5,7,0x21 | idle / standing (handles transition to 0x22 dead) |
| 1 | engaged (battle stance "ntbo") |
| 2,3 | sequence/sleep motion ("sped" when slept, bit 0xfe&1) |
| 6,0x26,0x27,0x28,0x29,0x2a,0x2b | **active/engaged combat states** — these 7 are the "casting/attacking" statuses that gate event NPC talk (also checked in RecvBattleCalc2 :11201) |
| 8,0x1a,0x1d | **door OPEN** (XiDoorActor::OpenDoor) |
| 9,0x1b,0x1e,0x2e | **door CLOSE** (XiDoorActor::CloseDoor) |
| 10-0x11,0x1c,0x1f,0x2c | no-op / static |
| 0x12-0x19 | **ship / airship / transport actions** (XiModelActor::SetShipAction) |
| 0x22 | **dead** (plays "into"/"note" death motion) |
| 0x2d | door open (variant, arg 1) |

The `+0xc2` "subkind" byte: 3 = door actor, 4/5 = special (ship), else normal PC/NPC.

**Server relevance:** `game_status` is LSB's entity **`status`** field
(`STATUS_NORMAL`=0, `STATUS_ENGAGED`=1, `STATUS_DEAD`=2/3, `STATUS_EVENT`=4,
`STATUS_OPEN_DOOR`=8, `STATUS_CLOSE_DOOR`=9, etc.). The mapping above is the
client-side authority for what each numeric status *plays*. Statuses
6/0x26-0x2b being the "busy" set explains why NPCs won't talk while in those
states.

---

## 4. EFFECT / ANIMATION / SOUND SCHEDULING

(See Section 6 for the schedule-script opcode table from the dedicated pass over
`ymschdecript.cpp` / `ymgenerater.cpp` / `ymsound.cpp`.)

Key client classes:
- `YmGenerater` (include/YmGenerater.h) — the particle/effect generator that
  *plays* a spell/status visual. Fields (DWARF/llm): `mCaster`/`mTarget`
  (+0x18/+0x24), `mDuration` (+0x8c), `mElapsed` (+0x88), `mLifeCount` (+0x54),
  `mFrame`/`mFrameStep`/`mFrameRate` (+0x90/+0x96/+0x98), `mGenerateNum` (+0x4c).
  These are the **timing fields that gate effect/animation length**. Code
  sub-types: `GenerateCode`, `InitCode`, `IdleCode`, `DieCode` (the 4
  per-effect code phases — GetCode/GetCodeName overloads, YmGenerater.h:75-78).
- `XiActor::SetAction / SetAttack / UseMagic` (XiActor.h:140,223-228) — invoked
  by the action packet handler to start the animation; they take a
  `CXiSchStatus*` (the unpacked action) or `(id, target, name, int, DAMAGE_KIND)`.
- Effect schedules are loaded by the event VM opcodes 0x2c/0x45/0x62/0x73
  (`SCHEDULOR`, `LOADSCHEDULER`, `LOADEVENTSCHEDULER`, `MAGICSCHEDULOR`).

`DAMAGE_KIND` is an enum (param to SetAttack/UseMagic) whose enumerators Ghidra
did not recover; it is the physical/magical/breath damage class.

---

## 5. ENUM / TYPE / CONSTANT CATALOG

There is no master `_types.h`. Named enums are forward-declared across headers
with **empty bodies** (Ghidra limitation). The gameplay-relevant ones and where
their *values* are actually decided:

| Enum | Declared in | Values decided in |
|------|-------------|-------------------|
| `GAME_STATUS` | XiAtelBuff.h:68 | xiatelnet.cpp:1848 (see §3.2) |
| `ACTOR_STATUS` | XiAtelBuff.h:71 | — |
| `SUBACTOR_STATUS` | XiActor.h:45 | IsChocobo/IsFishingRod (XiActor.h:83,246) |
| `CLI_STATUS` | XiAtelBuff.h:72 | — |
| `DAMAGE_KIND` | XiActor.h:224 | SetAttack/UseMagic |
| `RESULT_MES_TYPE` | CXiSchStatus.h:30 | PutMessageSP |
| `NAMECOL` | XiActor.h:247 | GetStatusNameColor |
| `CATEGORY` | KzResfList.h:20 | resource list |
| `USITEMNAMETYPE`, `USCAPTYPE` | xievent text fmt | dialogue bytecode (see §5.1) |
| `MENUCTRL_ID` (7×) | CTkMenuCtrlData.h | menu controls |

### 5.1 Dialogue text-format bytecode (XiAtelMess)

Distinct from the event VM: the dialogue string formatter
(xievent.cpp:1605-1796) interprets in-string control bytes when building NPC
text. Control opcodes (after 0x7f escape):

| Byte | Effect |
|------|--------|
| 0x80 | set CapType (capitalization) |
| 0x81 | set ItemNameType |
| 0x82 | inline number (literal) |
| 0x83 | number from EventPara[n] |
| 0x84 | expand bit-name list (`_BitName` table) |
| 0x85 | field separator |
| 0x89/0x8a | status message name (`_StatusMess` table, +0x100 variant) |
| 0x8b | number from CommMessWork[n] |
| 0x8c | atoi of string buffer |
| 0x8d/0x8e | event string from EventStrBuf (0x8e = +0x14 variant) |
| 0x8f | ability name (`_g_pKaDataAbility`, 0x30-byte stride) |
| 0x31-0x37 | raw copy 2 or 3 bytes (number/select tokens) |

**Server relevance:** LSB's NPC dialogue uses these same in-string format codes
(item/ability/number/player-name substitution). The DAT message holds the codes;
the params come from the event packet (`EventPara`, `EventStrBuf`).

### 5.2 Other constants worth noting

- Event char-request throttle: `_EventCharReqCounter = 0x78` (120 frames),
  xievent.cpp:2585 / 2450.
- Event WAIT default (op 0x6f): 16.0 frames, xievent.cpp:7836.
- Gosub stack depth: 8 (xievent.cpp:6930).
- Query window control: `CTkQueryControl::AddItem/AddComment/SetCursol`
  (xievent.cpp:3438-3464).

---

## 6. SCHEDULE-SCRIPT / SOUND OPCODES (ymschdecript / ymgenerater / ymsound)

The effect system is a **3-layer client-side VM**:
1. `ymschdecript.cpp` — the **schedule-script** VM (`YmSchedulerTask`) running
   per-action/per-spell scripts attached to caster/target actors.
2. `ymgenerater.cpp` — the **generator** VM (`YmGenerater`) that schedule
   opcodes invoke to spawn/animate particle/model/sound "elems".
3. `ymsound.cpp` — `YmSoundElem` / `YmSepRes` sound playback bound to those
   elems and to the UI.

The server only needs to send action/animation **IDs**; the client resolves them
through `document->vfunc(0x34)(type, id)` with resource types: **5**=generator/
effect, **7**=schedule, **6**=camera, **0x17**=motion, **0x3d**=sound (SEP),
**0xa**=texture, **0x3e**=path, **0x2f**=weather. **Animation length is
client-authoritative**: every duration is `speed_ratio * (ushort)tag[6]` frames
(60 fps), written into the spawned task's life counter. The script self-
terminates (`YmGenerater::IsNever`) and does NOT report completion back — so the
server's action timing must independently match these script durations.

### 6.1 Schedule-script control bytes (Stage A — `YmSchedulerTask::OnMove`, ymschdecript.cpp:399)

Dispatch on the tag's first byte as **ASCII char** (inner loop reads
`*(char*)tag` at :667):

| char | line | meaning |
|---|---|---|
| `'2'` 0x32 | :669 | advance to next status target (`CXiSchStatus::GetTarget`) |
| `'1'` 0x31 | :702 | idle-tag marker / jump to idle |
| `'='` 0x3d | :778 | **random branch**: scan to `'>'`(0x3e), pick `uirand(count)` |
| `':'` 0x3a | :828 | label / no-op |
| `'k'` 0x6b | :829 | **expression VM** (arith/compare on YmVariant stack) |
| `'j'` 0x6a | :1007 | **end / terminate** task |
| `'e'`/`'d'` | :1020 | **conditional fork**: pop int, fork if non-zero |
| `'R'` 0x52 | :1073 | **realtime gate** by XiDateTime game-clock ticks |
| `'a'` 0x61 | :1192 | **time-wait** until `time_cnt >= field*12.0` frames |
| `'{'` 0x7b | :1204 | **zone open / livecam** (`XiZone::Open`) |
| default | :1230 | fall through to Stage B `ExecuteTag` |

Expression opcodes (`'k'` tag, switch ymschdecript.cpp:853): 5=mul 6=div 7=add
8=sub; 9=shl 10=shr; 0xb=!= 0xc=== 0xd=>= 0xe=<= 0xf=> 0x10=<; 0x11=AND 0x12=OR;
0x14=set 0x15..0x18=+=/-=/\*=//=; 0x1c=push value; 0x1d=reset stack.

### 6.2 Schedule-script binary opcodes (Stage B — `YmSchedulerTask::ExecuteTag`, ymschdecript.cpp:1348, switch :1650)

Guard: opcodes > 0x8b rejected (:1647). Operands: `(ushort)tag[6]`=duration/
frames (×`speed_ratio`), `(u32)tag[8]`=resource id. (Ghidra mistyped the switch
selector as `float*`; read `case (float*)0x2C` as opcode **0x2C**.)

| op | action | op | action |
|----|--------|----|--------|
| 0x02 | **generate effect** (generator type 5; life=ratio×frames) | 0x03/0x73 | run sub-schedule |
| 0x04/0x81 | camera effect | 0x05 | set **caster motion** (type 0x17) |
| 0x06 | set **target motion** | 0x07/0x08 | bondage (constrain) caster/target |
| 0x09 | run schedule found on target | 0x0a/0x0b | **play SE on caster/target** |
| 0x0c/0x0d | door slide / door rotate | 0x0e | screen blur |
| 0x0f/0x51/0x72 | screen-color overlay / display / flash | 0x10 | screen capture overlay |
| 0x11-0x14 | caster/target Approach / BackJump | 0x15/0x16/0x22/0x23 | create doll-actor caster/target |
| 0x17/0x18 | retarget caster/target to its doll | 0x19 | set caster actor flag (vfunc 0x16c) |
| 0x1a | debug spawn actor | 0x1b/0x1c | set caster/target = found actor |
| 0x1d | elevator (lift) move | 0x1e/0x2d | **kill / deactivate generator** |
| 0x1f/0x20 | lock actor status caster/target | 0x21/0x25 | **damage motion** caster/target |
| 0x24 | **weaponskill/finisher motion select** (picks `itb*` res) | 0x26/0x27 | path-move caster/target |
| 0x28 | caster auto-run motion | 0x29/0x2a/0x46/0x47 | actor color tint caster/target |
| 0x2b | **put system message** (`PutMessage`) | 0x2c | after-image (motion blur) caster |
| 0x2e/0x2f | lock caster control / rotation | 0x30 | run schedule on **all** status targets |
| 0x3b/0x3c | run sub-schedule + suspend self | 0x3f | swap two generators |
| 0x40/0x41 | effect-texture caster/target | 0x42-0x45 | scrolling-texture effect |
| 0x48/0x49 | distortion (heat-haze) caster/target | 0x4a | play SE on target (conditional) |
| 0x53 | **play 3D SE on nearest target** | 0x54/0x55 | effect-texture w/ color |
| 0x56 | lock status on target/all | 0x57/0x85 | run/kill target-owned schedule |
| 0x58 | reset camera to default | 0x59 | lock caster magic |
| 0x5a/0x5b | guard/parry motion caster/target | 0x5c | run race-directional schedule |
| 0x5d | run 8-direction `ata*` schedule (dir via GetDirId_Div8) | 0x5e | **blowback / knockback** |
| 0x5f | kill all on schedule | 0x60 | **play system SE** (vol system×127) |
| 0x62 | turn caster to face target | 0x6e | set caster move speed |
| 0x6f/0x70 | point-light fade up/down | 0x71 | **put SP message** (`PutMessageSP`) |
| 0x74 | dissolve (disintegrate) target | 0x75 | show/hide weapon |
| 0x76/0x77 | run level-coded schedule caster | 0x78 | play SE by job/level-coded id |
| 0x79 | set caster field from id table | 0x7a | eye/link control |
| 0x7c | **set zone weather** | 0x7d | **set game time** |
| 0x7e | set default weather | 0x7f | animate specular caster |
| 0x80 | **use proc magic** (clones status) | 0x82/0x83 | DOF/focus change + restore |
| 0x84 | run weapon-coded schedule | 0x86 | set caster sub-actor flag |
| 0x87 | caster vfunc 0x16c | 0x88/0x89 | lock caster color / constrain |
| 0x8a | **play screen SE, store seid** | 0x8b | stop screen SE by stored seid |

**Server relevance:** opcodes 0x2b/0x71 render the battle text from
`CXiSchStatus` — message packet drives text + effect script in lockstep.
The whole table is fired *internally* by the client from action/spell ids; LSB
does not send these. They document what visual+sound a given spell/WS/ability
produces and how long it takes.

### 6.3 Generator VM (ymgenerater.cpp)

Four bytecode streams on each `YmGenerater` (record = byte0 opcode, byte1&0x1f =
length in 4-byte words, opcode 0 terminates; `GetCode` ymgenerater.cpp:7589+):
- **+0xa0 GenerateCode** (spawn/tracking), **+0xa4 InitCode** (per-elem init),
  **+0xa8 IdleCode** (per-frame anim), **+0xac DieCode**. (The `GenerateCode`/
  `InitCode`/`IdleCode`/`DieCode` operand-type enums are *DWARF-authoritative*.)
- **IdleCode** opcodes (`ElemIdle`, switch ymgenerater.cpp:1466, cases 0x01-0x6e):
  per-frame position/rotation/scale/color/alpha/UV keyframe mutators (each ×Δt
  `iRam00557a44`). 0x1=vel→pos, 0x4-0x6=rotation (angle-wrap via fmodf),
  0x7-0x9=scale, 0xa/0xb=color add.
- **InitCode** opcodes (`ElemGenerate`, switch ymgenerater.cpp:4483, cases
  0x01-0x9e — the largest opcode set): 0x1 = create child model/elem (life from
  `tag[0x20]`); rest set initial pos/dir/scale/color/blend/texture/**life/frame
  counts**/billboard/child-generator links; binds a `YmSoundElem` when the model
  is a `YmSepRes`.
- Lifecycle: `Activate` :996, `Deactivate` :1054, `ElemDie` :1240, `KillAll`
  :7823, `IsNever` :10806 (effect-finished test). `GetModelType` :8108
  (0x3d=sound, 0x47=special).

### 6.4 Sound (ymsound.cpp)

**Volume categories** (`VOLUMEATTR`, ymsound.cpp:54): **1=System, 2=Effect,
4=Zone** (3 unused); Master = all three. Timed fades via `YmVolumeChangeTask`.
(These are the same 4 bits the event opcode 0x69 toggles, §1.3.)

**Sound-id resolution:** the playable id is the SEP resource's `Sequence`
(`YmSepRes::sep_header.Sequence`). `YmSoundElem::OnPlayUpdate` (:903) does 3D
pan/vol via `Calc3D` then `SqsSePlay(1, Sequence, pan, vol)`. 3D falloff uses
`sound_near`/`sound_far`/`sound_width`; pan clamped, vol baseline +0x40, clamped
0..0x7f. `time_margin = 3.0` frames keepalive.

**Music:** `YmMusicServer::Play(num, vol, fade_in, fade_out, replay_point)`
(:1780); fade-in clamped to min 30 frames.

**UI SE number table** (each = resource type **0x3d** + 4-char ASCII key, system
volume, pan 0x40):

| function | SE key | function | SE key |
|---|---|---|---|
| YmSePlayCursor | "1000" | YmSePlayClick | "2000" |
| YmSePlayCancel | "3000" | YmSePlayBeep | "4000" |
| YmSePlayWarning | "5000" | YmSePlayError | "6000" |
| YmSePlayTarget | "9000" | YmSePlayTargetCursor | "0100" |
| YmSePlayWindowSelect/OpenMainWindow/MessageFeed/ScreenShot | "01x0" range | YmSePlayChime | dynamic (power-coded) |

**Server relevance:** these UI SE ids are client-local (menu sounds). The
combat/effect SE ids come from the SEP resources referenced by the schedule-
script opcodes (0x0a/0x0b/0x53/0x60/0x78/0x8a) and are selected by the action
id, not sent explicitly by the server.

---

## 7. QUICK INDEX OF KEY FILE:LINE ANCHORS

- Event VM dispatch: `src/main/actor/xievent.cpp:6646` (`ExecProg`)
- Event menu build: `xievent.cpp:3350` (`CodeQUERY`)
- Event result send (**packet 0x5b**): `src/main/actor/xiatelnet.cpp:12572`
- Event receive: `xiatelnet.cpp:12276 / 12373 / 12419`
- Event load: `xievent.cpp:2416` (`InitEvent`, files 0x16bc+N / 0x1914+N)
- Action packet unpack: `src/common/XiSchStatus.cpp:351` (bit layout :456+)
- Action dispatch: `xiatelnet.cpp:11171` (`RecvBattleCalc2`, switch :11224)
- Action field names: `include/BattleResult.h`, `include/CXiMainToCalc.h`
- Status icons (**64 u16**): `xiatelnet.cpp:9282` (`RecvCliStatus2`)
- Buff cancel (**packet 0x0f1**): `xiatelnet.cpp:12495`
- GAME_STATUS → motion: `xiatelnet.cpp:1848` (`SetStatusMotion`)
- Schedule-script VM: `src/main/miyagawa/effect/ymschdecript.cpp:1348` (`ExecuteTag`)
- Generator VM: `src/main/miyagawa/effect/ymgenerater.cpp:4483 / 1466`
- Sound: `src/main/miyagawa/ymsound.cpp:903 / 1780 / 2210+`
