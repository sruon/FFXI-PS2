# FFXI PS2 Client — Mechanics & Protocol Findings

A multi-agent deep review of the decompiled FFXI PlayStation 2 client
(`SCUS_972.66`, MIPS R5900, 2003) cataloging **gameplay mechanics encoded in
the client** and **everything that interfaces with the server**, written for a
server-emulator (LandSandBoat / "LSB") audience.

Each finding is tagged for provenance: **[DWARF]** authoritative symbol/field
names from debug info, **[CODE]** literal value present in the decompiled
source, **[RODATA]** value lives in the binary's data segment and is *not* in
this decomp (only the `DAT_*`/address is — needs a `.rodata` extraction pass),
**[INFER]** reasoned from access patterns.

## Documents

| # | File | Scope |
|---|---|---|
| 01 | [`01-combat-action-mechanics.md`](01-combat-action-mechanics.md) | Target ranges, damage caps, action categories, collision, buff-locks |
| 02 | [`02-incoming-packets.md`](02-incoming-packets.md) | Server→client dispatch table, framing, actor/inventory/chat structs |
| 03 | [`03-outgoing-packets-wire-protocol.md`](03-outgoing-packets-wire-protocol.md) | enAcvSet pipeline, modified blowfish, action packet 0x1A, throttle |
| 04 | [`04-world-actor-simulation.md`](04-world-actor-simulation.md) | Entity-ID scheme, heading/movement constants, lifts/doors, position reporting |
| 05 | [`05-ui-data-systems.md`](05-ui-data-systems.md) | Inventory/equip caps, item flags, party/AH/trade/combine/chat limits |
| 06 | [`06-effects-events-enums.md`](06-effects-events-enums.md) | Event bytecode VM, action-packet bit layout, GAME_STATUS, status icons |
| 07 | [`07-login-auth-telemetry.md`](07-login-auth-telemetry.md) | Lobby handshake, MD5 key ratchet, command-id map, bug telemetry |

---

## The five things to read first

### 1. This is the 2003 PS2 protocol, NOT retail PC — opcodes differ
Packet fragment IDs are **9-bit** (`id = word & 0x1ff`), dispatched through an
80-entry table (`02 §dispatch`). There is **no 0x28 action / 0x29 basic-message
packet** here. Combat/system text rides `0x09 MESSAGE`; actor state rides
`0x0D/0x0E`. **LSB's modern retail packet map does not apply 1:1** — the DWARF
struct type names (`GP_SERV_*`) in doc 02 are the authoritative reference for
this client. This is the single most important caveat for anyone diffing
against a retail-era server.

### 2. The client does almost no game-rule validation — the server is fully authoritative
Outgoing actions (`03 §6`, `01 §5`) clamp only **IDs**: magic/weaponskill ≤
`0x1ff`, ability ≤ `0x2ff`, item index 1–`0x50`, spell 0..0x1ff. There are **no
client-side range / level / recast / MP / line-of-sight checks** before send.
Everything gameplay-meaningful (distance, resists, costs, cooldowns) **must** be
enforced server-side; the client values below are *advisory* and patchable.

### 3. Wire format an emulator must reproduce exactly (`03 §1-4`, `07 §A-B`)
- **Game datagram** = 0x1c-byte cleartext header (`u16 MyCnt seq`, `u16 YouCnt
  ack`, `u32 ms`, `u32 sec`) + concatenated 9-bit-ID sub-packets, each with its
  own 4-byte header and 16-bit seq; reliable resend until the server echoes.
- **enAcvSet pipeline**: `blowfish( huffman(payload) || bitlen(4B) || MD5(16B) )`
  — MD5 over the *compressed* bytes, then encrypt; bad MD5 = dropped.
- **Modified FFXI Blowfish**: F-function S1/S3 outputs masked `& 1 ^ 0x20`
  (collapsed to 0x20/0x21) — *not* textbook Blowfish. 16-byte per-session key,
  **rotates mid-session** (`enAcvChange`).
- **Static Huffman table** with raw-fallback flag (`out[0]` = 1 coded / 0 raw).
- **Lobby** is separate: `"FFXI"` (`0x46465849`) magic + whole-packet MD5 in the
  header, a **stateful MD5 key ratchet** (`lpkt_work+0x138`), 28–4031 byte size
  window, 0x8c-byte character records.

### 4. Hard constants baked into the client
See the table below — these are real `[CODE]` literals (vs `[RODATA]` numbers
that still need a binary dump).

### 5. The action result & event packets are bit-packed and client-decoded
- **Action result** (`06 §4`, `CXiSchStatus::Unpack`): LSB-first bitstream —
  actor(32) targetcount(6) category(4) info(4) param(32); per-target id(32)
  resultcount(4); per-result reaction(2) anim(2) effect(10) message(4) scale(5)
  **value(16)** plus flag-gated additional-effect & spike blocks (each
  6/4/14/9). **Damage value is 16 bits → hard display cap 65,535**; add-effect
  and spike damages are 14-bit (≤16,383).
- **Event** (`06 §1-3`): inbound `GP_SERV_EVENT/EVENTNUM/EVENTSTR`; the client
  runs a ~136-opcode **event bytecode VM** (`XiEvent::ExecProg`); player menu
  choice goes back as **packet 0x5b** (option index in the +0x08 result field,
  cancel = `0x40000000`). LSB event scripting must match these opcodes/params.

---

## Hardcoded gameplay constants (quick reference)

| Value | Meaning | Source | Tag |
|---|---|---|---|
| `2500.0` (=50²) | Max target-select range, **3D** (height counts) | `xiactor.cpp:1074-1660` | [CODE] |
| `36.0` (=6²) | Trade / interaction proximity | `TkTrade.cpp:1835` | [CODE] |
| `64.0` (=8²) | Player-collision contact radius | `xicontrolactor.cpp:1574` | [CODE] |
| `2500.0` (=50²) | Targeting reach (world layer) | `04 §5` | [CODE] |
| `30.0` | Movement speed cap (units/frame, pre-/60) | `xicontrolactor.cpp:631` | [CODE] |
| `× 2.0` | Chocobo (GameStatus==5) speed multiplier | `xicontrolactor.cpp` | [CODE] |
| `0.25` | Backwalk speed factor | `04 §3` | [CODE] |
| `1.5` | Walk→run animation ratio cutoff | `04 §3` | [CODE] |
| GM ≥ 3 | Disables player collision (no-clip) | `xicontrolactor.cpp:1574` | [CODE] |
| 16 bits | Action-result damage field → cap **65,535** | `XiSchStatus.cpp:621` | [DWARF/CODE] |
| 14 bits | Additional-effect / spike damage → cap 16,383 | `XiSchStatus.cpp` | [CODE] |
| `0x1ff`/`0x2ff` | Client ID clamps: magic·WS / ability | `commandcalc.cpp:3429+` | [CODE] |
| 81 / 3 | Inventory slots per container / container count | `KmItem.h:23`, `gcitem.c:53` | [DWARF/CODE] |
| 16 | Equipment slots | `gcequip.c:32` | [CODE] |
| `0x8000`/`0x4000` | Item flag bits: RARE / EX | `TkItemInfo.cpp:364` | [CODE] |
| 6 / 18 | Party / alliance member caps | `ykWndParty.cpp:835` | [CODE] |
| 7 (8 stored) | Active auction sells (likely off-by-one in client) | `ykWndAuction.cpp:1559` | [CODE] |
| 150 (`0x96`) | Chat body cap, all channels; **no send rate-limit** | `gcchat.c:112` | [CODE] |
| 100 / 300 | Blacklist / POL friends caps | `05 §10` | [CODE] |
| 1/256 | Lift floor-height fixed-point precision | `xiliftactor.cpp:227` | [CODE] |
| 28–4031 | Lobby packet size window | `nttcpmakepacket.cpp:54` | [CODE] |

---

## Encodings worth knowing (often mismodeled by emulators)

- **Heading is radians, not a 0–255 byte, in the client core** (`04 §2`):
  `dir = -atan2f(dz,dx)`. The byte form exists *only on the wire*
  (`enDirCliToNet`: `byte = (-radians)·128/π`; sign flipped). Decode accordingly
  server-side.
- **Position on the wire** = `int(coord·1000)` + 1-byte direction (`03 §9`).
- **Client reports collision-resolved (snapped) positions**, polled on a
  staggered ~2s cadence when stationary / full-rate when moving (`04 §9`).
  Server distance/anti-cheat checks should expect snapped, not raw, positions.
- **Entity IDs**: `id & 0xFF000000 == 0` ⇒ PC, else NPC/mob (`04 §1`). The
  targid(0–1023) table lives in the atel layer, not the actor list.
- **Action category enum** (client→server packet `0x1A`, `01 §4` / `03 §5`):
  2=engage, 3=cast, 4=disengage, 5=/heal, 7=weaponskill, 9=ability, 0xc=assist,
  0xf=switch target, 0x10=ranged, 0x11=dig, 0x12=dismount, 0=NPC-talk.

---

## Notable open items / caveats

- **The "8-yalm vertical AOE height limit" was NOT found** in this build's
  combat/effect code (`01` top note). AOE target-set selection appears to be
  driven by the server here — the client only sends the primary target's id and
  decodes a server-supplied per-target result list. (Per maintainer: this rule
  may be beta-version-specific and absent from this client.)
- **Movement tuning floats are `.rodata`-only** (`04` final section): turn rate,
  strafe speed, collision-radius scale, slope limits, the knockback table
  (`BlowBackParaTab`, 8×{power,dumper,timer}) are referenced by address but
  their literal values are not in this decomp. Recovering them needs a second
  pass dumping `SCUS_972.66`'s data segment.
- **POL message channel** (`02 §10`, `gcpolmessage.cpp`, 298 KB): auction/mail/
  friends use a separate `FFPMsg_Ctrl` opID scheme, *not* the zone 9-bit IDs —
  not opcode-mapped in this review; a worthwhile follow-up.
- **Schedule-script VM** drives **client-authoritative animation length**
  (`06 §8`): `speed_ratio × tag[6]` frames @60fps. Server action timing must
  independently match or animations/locks desync.
