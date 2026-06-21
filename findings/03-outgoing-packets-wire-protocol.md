# FFXI PS2 Client — Outgoing Packets & Wire Protocol

Scope: what the retail PS2 client (SCUS_972.66, image version "972", `image_base=0x00100000`) **sends** to the
game server, the on-the-wire framing/encryption an emulator must replicate, and the client-side gating around
each request. Source is decompiled C/C++ with recovered DWARF symbols (`ffxi-decomp`, ghidra-12.0.4).

**Naming authority:** Function names quoted from the `// symbol:` header comments are DWARF-authoritative
(e.g. `gpGamePacketSet`, `enAcvSet`, `makeActionPacket`, `gcEquipChangeSet`). Struct *field* names are
**inferred** from offsets/usage unless stated otherwise — Ghidra emitted raw `*(type *)(ptr + off)` accesses,
not named members. Offsets are quoted exactly; field meanings are inferred and flagged where uncertain.

> Terminology used below: a **queue slot** is the 0x110-byte staging buffer returned by
> `gcZoneSendQueSearch(PacketID, …)`. Byte **+0** is the packet type ID (low 9 bits), byte **+1** carries the
> size, and builders write payload starting at **+4**. Multiple slots are later concatenated by
> `gpGamePacketSet` into one UDP datagram. See §1.

---

## 1. Wire protocol — framing, sequence, checksum, compression, encryption

The send path is: per-action builders fill a **queue slot** → `gpGamePacketSet` concatenates all pending slots
into a datagram body → `enAcvSet` huffman-compresses + appends MD5 + blowfish-encrypts → `ntUdpSend*` transmits.

### 1.1 Datagram header — `gpGamePacketSet`
`src/main/net/game_prot/gpgame.c:17` (DWARF `gpGamePacketSet`, rva 0x00087af0).

```
gpGamePacketSet(pSend, pQueSys, MyCnt, YouCnt, CheckCnt, table)
  *pSend            = MyCnt;                 // +0x00 u16  outgoing packet counter (our seq)
  pSend[1]          = YouCnt;                // +0x02 u16  ack of last received server seq
  *(u32*)(pSend+2)  = ntTimeNowGetMSec();    // +0x04 u32  client timestamp (ms)
  *(u32*)(pSend+4)  = ntTimeNowGetSec();     // +0x08 u32  client timestamp (s)
  __dest = pSend + 0xe;                       // payload region begins at +0x1c
```
- **Header is 0x1c (28) bytes.** Payload (concatenated sub-packets) starts at offset 0x1c. (`enAcvSet`
  copies exactly `0x1c` as the cleartext header — gpgame.c:36 `iVar4=0x1c; iVar5=0x1c;`, enacv.c:80
  `memcpy(pCode,pText,0x1c)`.)
- **MyCnt / YouCnt are 16-bit sequence numbers**, incremented in `SendProc` (`gczone.c:1812`:
  `*(short*)(pStack+6) += 1; if == 0 then +=1` — seq never zero). YouCnt is the last-acked server count.
- Each concatenated **sub-packet** has its own 4-byte header: byte0 = 9-bit type ID (`*src & 0x1ff`,
  gpgame.c:48), byte1 = size in 32-bit words (`(b>>1)&0x7f`, i.e. `size*4`), bytes2-3 = a per-packet
  sequence stamp (`__src[1] = MyCnt` when first sent, gpgame.c:81).
- **Reliable resend:** sub-packets stay in the ring queue and are re-sent until acked. gpgame.c:54
  `if (__src[1]==0 || 0x7fff < (CheckCnt - __src[1]))` decides resend vs. drop-when-acked. **Server-relevance:**
  the server must echo the client's MyCnt back as its YouCnt (and vice-versa) so the client can retire
  acked sub-packets; otherwise the client resends forever. LSB's session already implements this
  (`PacketParser`/`PrintPacket` seq fields) — confirm 16-bit wrap and "never zero" handling.
- **MTU / size caps:** a datagram body caps near `*(pGame+0x458)-0x31` bytes and the uncompressed total caps
  at 3999 bytes (`gpgame.c:58-59`). The reassembled cleartext output buffer in `enAcvSet` is `0x578`
  (gczone.c:1790/1797) ≈ 1400 bytes. **Server-relevance:** outbound (server→client) packets must also
  respect these limits; LSB already chunks at ~1400.

### 1.2 Compression + checksum + encryption — `enAcvSet`
`src/main/net/engine_net/enacv.c:57` (DWARF `enAcvSet`, rva 0x00086ba0). This is the *single* function that
turns a cleartext datagram into the on-wire blob. Order of operations (exactly):

1. **Copy the 0x1c header verbatim** (cleartext, not compressed): `memcpy(pCode, pText, 0x1c)` (enacv.c:80).
2. **Huffman-encode the payload** (everything after 0x1c): `huffman_packet_encode(...)` (enacv.c:87). When
   `pAcv==0` (pre-zone, see §1.5) the payload is copied **raw** instead (enacv.c:84).
3. Append the **encoded bit-length** as a 4-byte little-endian int right after the compressed stream
   (enacv.c:100 `memcpy(__dest + (bits>>3), &iStack_64, 4)`).
4. **MD5 over the (header-excluded) payload+length region**: `fs_MD5Init/Update/Final`, then
   `memcpy(__dest + len, md5, 0x10)` (enacv.c:114-117). **16-byte MD5 digest is appended.**
5. **Blowfish-encrypt** payload+length+digest (NOT the 0x1c header) when `pAcv!=0`:
   `blowfishEncryptString(pAcv, __dest, len + 0x10)` (enacv.c:119).
6. Returns total size = `len + 0x2c` (0x1c header + 0x10 MD5 region accounting, enacv.c:121).

`enAcvGet` (enacv.c:136) is the inverse for inbound: blowfish-decrypt, recompute MD5 over `len-0x2c` and
`memcmp` against the trailing 16 bytes (enacv.c:181) — **packets failing the MD5 check return 0 (rejected)**.
Bounds: encoded length must be `0x2d..0x578` (enacv.c:166-167).

**Server-relevance (critical):** an emulator must, for every game-channel packet:
the cleartext 0x1c header travels in the clear; the rest is `blowfish( huffman(payload) || encoded_bitlen(4) || MD5(huffman||len)(16) )`.
MD5 is over the *compressed* bytes plus the 4-byte length, *before* encryption. This matches LSB's
`md5` + `blowfish` usage in `zlib.cpp`/`blowfish.cpp` — but note FFXI's blowfish is **non-standard** (§1.3).

### 1.3 Blowfish — MODIFIED variant
`src/main/net/engine_net/blowfish.c` (DWARF `newBlowfishKey`, `blowfishEncryptString`, `blowfishDecryptString`).

- **Key schedule** `newBlowfishKey` (blowfish.c:16): standard P-array (`P_orig`) and 4×256 S-boxes
  (`S_orig`), keyed by repeating the key bytes; 521-iteration warmup. Standard Blowfish *structure*.
- **F-function is DELIBERATELY WEAKENED.** In both encrypt and decrypt, the two S-box lookups that feed
  the high byte and the bit-8 byte are masked:
  ```c
  // blowfish.c:101-104 (and repeated for all 16 rounds)
  (keyStructure[(x>>0x18)+0x312] & 1 ^ 0x20)            // S3[ x>>24 ]  → forced to 0x20 or 0x21
  + keyStructure[((x&0xff0000)>>0x10)+0x212]            // S2[...]      (normal)
  + keyStructure[(x&0xff)+0x12]                          // S0[...]      (normal)
  + (keyStructure[((x&0xff00)>>8)+0x112] & 1 ^ 0x20)    // S1[...]      → forced to 0x20 or 0x21
  ```
  So S1 and S3 outputs are collapsed to a single bit (0x20 or 0x21) instead of a full 32-bit word. This is
  SE's custom FFXI blowfish, NOT textbook Blowfish.
- 16 Feistel rounds, final block swap, then XOR with `P[16]`/`P[17]` (blowfish.c:188-192). Operates on
  8-byte blocks; `length>>3` blocks processed (blowfish.c:99) — **trailing bytes that don't fill an 8-byte
  block are left unencrypted** (the MD5+length padding guarantees the encrypted region is a multiple of 8).
- The Blowfish KEY is derived from a 0x50-byte password/key blob in `enAcvCreate`
  (`src/main/net/engine_net/enacv.c:15`): `strncpy(pAcv+0x1048, pPass, 0x50)`, then
  `newBlowfishKey(pAcv, 0x1048, pAcv+0x1048, 0x10)` — **a 16-byte (0x10) key**. The key originates from the
  zone/login handshake. **Server-relevance:** LSB must use this exact modified F-function (it already does in
  its `blowfish.cpp` for the PC client — verify the S1/S3 `&1^0x20` masking is present; this is the same
  weakened variant the PC client uses). The 16-byte per-session key is delivered during zone-in (§3, packet 0x0A).

### 1.4 Huffman — static table, with raw fallback
`src/main/net/engine_net/huffman.c` (DWARF `huffman_packet_encode`, `_decode`, `newHuffmanTable`, `newHuffmanTree`).

- **Static, fixed code table.** Encode table built from global `Huffman_Buffer` (huffman.c:27-28); decode
  tree from `0x4dc5e0` / `HuffmanDat` size 0x900 (huffman.c:49). 256 symbols (one per byte value).
- **First output byte is a flag.** `huffman_packet_encode` sets `resultBuffer[0]=1` for huffman-coded data
  (huffman.c:289) and `resultBuffer[0]=0` for the **raw fallback** used when compression doesn't fit
  (huffman.c:300-306, copies `buffer` verbatim after the flag). `huffman_packet_decode` branches on
  `codedBuffer[0]==1` (huffman.c:157) vs raw memcpy (huffman.c:176).
- Code length per symbol is precomputed (`huffman_packet_encoded_length`, huffman.c:248, table at +0x400);
  symbol index is `(byte + 0x80)` (signed-char offset, huffman.c:257/292).
- **Server-relevance:** an emulator MUST ship the identical static huffman table (same as the PC client's
  fixed table). Honor the leading flag byte: bit set = huffman stream, clear = raw. LSB already does this.

### 1.5 Encrypted vs. cleartext phases
`SendProc` (gczone.c:1788) chooses the mode by `*(byte*)(pZone+3)` (connection state):
- state `< 2` (pre-zone / initial handshake): `enAcvSet(..., pAcv=0, ...)` (gczone.c:1790) → **no blowfish,
  no huffman, raw payload** with MD5 still appended. First byte `uStack_2714=0`.
- state `>= 2` (zoned in): `enAcvSet(..., pAcv = pZone+0xc32, ...)` (gczone.c:1797) → **full blowfish +
  huffman**. First byte `uStack_2714=1`; also writes two key/seq dwords `pZone[0x105b]/[0x105c]`.
- **Key rotation:** `enAcvChange` (enacv.c:40) bumps a counter `pAcv+0x109c` every time the client's local
  send-key index matches the negotiated one (gczone.c:1817-1820). **Server-relevance:** the blowfish key is
  rotated/re-derived during the session; LSB tracks this via the "key" returned in zone packets.

### 1.6 MD5
`src/main/net/engine_net/fs_MD5.c` — standard RFC-1321 MD5 (init constants `0x67452301 …` at fs_MD5.c:20-23).
`createLoginTicket` (enacv.c:223) = `MD5(accountName || key)` → 16-byte login ticket (used in zone-in, §3).

---

## 2. The action request — packet 0x1A (the most important client→server packet)

This is the universal "do something to a target" request (cast, attack, use ability/WS/item, engage, etc.).
**DWARF-authoritative builder:** `makeActionPacket` — `src/main/actor/commandcalc.cpp:1693`
(`makeActionPacket__FUiUiUsUi`, rva ~0x00183xxx).

```c
makeActionPacket(ActIndex, UniqueNo, ActionID, ActionBuf):
  slot = gcZoneSendQueSearch(0x1a, 0, 0);
  *(u16*)(slot + 8)  = ActIndex;     // +0x08 u16  target's actor/activity index (server "TargIndex")
  *(u32*)(slot + 4)  = UniqueNo;     // +0x04 u32  target's UniqueNo (server entity ID)
  *(u16*)(slot + 10) = ActionID;     // +0x0a u16  action CATEGORY (see table)
  *(u32*)(slot + 12) = ActionBuf;    // +0x0c u32  PARAM (spell id / ability id / WS id / etc.)
  gcZoneSendQueSet(slot, 0x10, 0);   // total 16 bytes
```

**Struct (16 bytes / 0x10):**

| Offset | Size | Field (inferred) | Meaning |
|--------|------|------------------|---------|
| +0x00 | u16 | packet ID | 0x1A (auto) |
| +0x04 | u32 | UniqueNo | target server entity ID |
| +0x08 | u16 | ActIndex | target index (the short-form target id) |
| +0x0a | u16 | ActionID / **Category** | what to do (table below) |
| +0x0c | u32 | ActionBuf / **Param** | spell/ability/WS id or sub-param |

**Action category values** (all from `commandcalc.cpp`, `*(u16*)(slot+10) = N`; command names are DWARF
function names of the `cmdf_COM_*` handler that emits them):

| Category | Source line | Emitting command / meaning |
|----------|-------------|----------------------------|
| 0x00 | 3748 | (target-only / interact) |
| 0x02 | 2842, 2880 | engage-related / attack target |
| 0x03 | 3446 (`cmdf_…`, param = spell id, clamped `0..0x1ff`) | **cast magic** (param = spell ID) |
| 0x04 | 2936, 2976 | check / target action |
| 0x05 | 3266 | |
| 0x06 | 10418 | |
| 0x07 | 3519 | |
| 0x09 | 3586 | |
| 0x0c | 3064 (`cmdf_COM_ASSIST`) | **assist** |
| 0x0e | 3207 | |
| 0x0f | 2908, 3025 | |
| 0x10 | 2751 (`cmdf_COM_ATTACK_toggle`) | **engage / attack on** |
| 0x11 | 3149 | |
| 0x12 | 3103 | |
| 0x0a (10) | 10466 (param = `lo & 0xff | (hi&0xff)<<8`) | two-byte sub-param action |

> These category numbers correspond to the well-documented FFXI 0x01A "action" subcommands (3=magic,
> 7=weaponskill, 9=ranged, 0x0c=assist/check, 0x10=engage, 0x12=dismount, etc.). The exact mapping above is
> inferred from the emitting `cmdf_COM_*` names where present; treat unnamed rows as "category N, param at +0xc".

**Server-relevance:** This is LSB's `GP_CLI_COMMAND` / `0x01A` action packet. Field layout the server reads:
UniqueNo (+4), TargIndex (+8), Category (+0xa), Param (+0xc). **The client clamps spell id to `0..0x1ff`
(commandcalc.cpp:3429) but performs essentially NO other validation** — range/level/recast/distance checks
are NOT done client-side, so the server MUST enforce them independently. Note 0x1A has **no per-command
rate-limit** in `gcZoneSendQueSearch` (`SearchSub=0` → bypasses the throttle table, see §6).

---

## 3. Zone / session-control packets (gczone.c)

| ID | Builder (DWARF) | Size | Key fields | Notes / server-relevance |
|----|-----------------|------|------------|--------------------------|
| 0x0A | `StepCalc` (gczone.c:2261) | 0x5c | +0x04 checksum u8; +0x0c char-id u32; +0x22 char name[15]; +0x31 account name[15]; +0x40 **login ticket (16B MD5)**; +0x50 client version; +0x58 lang; +0x5a region; ticket = `createLoginTicket(name,key)` | **Zone-in / login handshake.** Sent pre-encryption. Carries the MD5 login ticket the server validates. Checksum at +4 = sum of bytes [8,0x5c). |
| 0x0B | `gcZoneLogOut` (1248) / `gcZoneChangeSet` (1283) | 0x1c | subtype @+0x18 (1=logout, 2=zonechange); zonechange also writes x/y/z*1000 @+4..+0x14, dir @+0x1b (`enDirCliToNet`) | Logout & zone-transition request. Position is **int(coord*1000)** + 1-byte dir. |
| 0x0D | `…` (gczone.c:2232) | 8 | +4 u16 = 0 | Client-ready / zone-confirm ack. |
| 0x0F | `gcZoneSendClientGameStatus` (1491) | 0x24 | +4 = `ntGameGetStatus(…,0x20)` 32-byte status blob | Periodic client game status. |
| 0x10 | `StepCalc`-adjacent (10, gczone.c:2274) | 0x5c | char-id @+0xc; name @+0x22; account @+0x31; version @+0x50; lang @+0x58; ticket @+0x40 | The actual login/connect packet (built when `pZone[0xc]==1`). Mirrors 0x0A layout. |
| 0xF0 | `gcZoneRescueSend` (1457) | 8 | +4 u32 State | "Rescue"/unstuck request. |
| 0xF2 | `gcZoneSubMapChangeSet` (1332) | 8 | +4 u16 State; +6 u16 SubMapNumber | Sub-area / map-region change. |
| 0x17 | `DebugReqCharSend` (2344) / `RecvDebugPc` | 0x14 | +4 ActIndex u16; +8 UniqueNo2 u32; +0xc UniqueNo3 u32; +0x10 Flg u16; +0x12 Flg2 u16 | Request char/PC data (widescan-style / appearance request). |

**Position reporting:** the player's authoritative position is held in zone state at `_pGlobalNowZone+0x25140`
(x), `+0x25144` (y), `+0x25148` (z), `+0x2514c` (dir) — set via `gcZonePosSet` (gczone.c:818). It is packed
into the zone-change packet 0x0B as `int(coord*1000)` and a 1-byte network direction
(`enDirCliToNet`, encalc.c:34: `dir_net = (dir_cli * 256) / 2π`). **Server-relevance:** distance/speed checks
must convert the 1-byte dir (0-255 = full circle, `enDirNetToCli`, encalc.c:56) and treat coords as floats.
The continuous per-tick position update (FFXI 0x015/0x016 "standard client" on PC) is emitted by the same
SendProc/queue path; on this build position rides the standard sub-packet stream rather than a dedicated
`gcZoneSendQueSearch(0x15)` builder — no 0x15/0x16 *builder* exists in game_cli (confirmed by grep).

---

## 4. Action-request packet builders by subsystem

Offsets are within the queue slot (+0 = id, payload from +4). "Size" = bytes passed to
`gcZoneSendQueSet`. All builders require `_pGlobalNowZone != 0`. Files under
`src/main/net/game_cli/`.

### 4.1 Equipment — gcequip.c
| ID | Builder | Size | Fields | Server reads / validation |
|----|---------|------|--------|---------------------------|
| 0x50 | `gcEquipChangeSet` (135) | 6 | +4 u8 ItemIndex (inv slot; 0=unequip); +5 u8 EquipKind (slot) | Equip request. **No client clamp** on slot/index. |
| 0xDD | `gcEquipStartInspect` (223) | 0xC | +4 u32 UniqueNo; +8 u32 ActIndex | "Check" a target's gear. |
| 0xDE | `gcInspectMessageSet` (159) | 0x80 | +4 char[123] message | Bazaar/inspect comment; client caps at 123 chars. |
| 0xE0 | `gcUserMessageSetWithType` (496) | 0x98 | +4 msg[128]; +0x84 install-time; +0x88 4-char tag; +0x8b install-size; +0x8c srv-excode; +0x90 cli-excode; +0x94 u32 type | Search-comment / user message + version metadata. |

**Equipment graphics validation** `gpGrapCheck` (`src/main/net/game_prot/gpequip.c:15`, DWARF) validates a
9-entry model table the client renders and **silently clamps** out-of-range values rather than rejecting:
race low-byte must be `1..0x1f`; face/hair high-byte `1..0x1f`; for the 8 model slots, top nibble must match
the per-slot "kind" in `GP_GRAP_KIND_DATA` and the low 12 bits must be `< max_model_id`. **Server-relevance:**
appearance/model IDs the server *sends* must fit these ranges or items render as "no model" client-side.

### 4.2 Inventory & items — gcitem.c
| ID | Builder | Size | Fields |
|----|---------|------|--------|
| 0x28 | `gcItemDumpSet` | 0xC | +4 u32 Num; +8 u8 Cate(bag); +9 u8 ItemIndex — **drop/dump item** |
| 0x29 | `gcItemMovSet` | 0xC | +4 u32 Num; +8 u8 srcCate; +9 u8 dstCate; +0xa u8 srcSlot; +0xb u8 dstSlot — **move between containers** |
| 0x2A | `gcItemAttrReqSet` | 6 | +4 u8 Cate; +5 u8 Index — request item attributes |
| 0x32 | `gcItemTradeReqSet` | 0xC | +4 u32 UniqueNo; +8 u16 ActIndex — **start trade with player** |
| 0x33 | `gcItemTradeStartSet`/`Cancell`/`Make`/`MakeCancell` | 0xC | +4 u32 action (0=offer,1=cancel,**2=confirm**,**3=un-confirm**); +8 u16 partner ActIndex |
| 0x34 | `gcItemTradeMyListSet` | 0xC | +4 u32 ItemNum; +8 u16 ItemNo(catalog); +0xa u8 inv slot; +0xb u8 trade slot(0-9). **Client rejects same inv slot placed twice.** |
| 0x37 | `gcItemUseSet` | 0x10 | +4 u32 UniqueNo; +8 u32 =0 (reserved); +0xc u16 ActIndex; +0xe u8 ItemIndex — **use item on target** |
| 0x38 | `gcItemDebugMakeSet` | 0xC | +4 u32 Num; +8 u16 ItemNo — **GM/debug create item** (do not implement for retail) |
| 0x39 | `gcItemListReqSet` | 4 | header-only — request full item list (also zeroes local cache) |
| 0x3A | `gcItemStackSet` | 8 | +4 u32 Category — sort/stack a container |
| 0x41 | `gcItemTrophyEntrySet` | 6 | +4 u8 TrophyIndex; +5 u8 ItemIndex (auto-resolved if 0) — place furniture |
| 0x42 | `gcItemTrophyAbsenceSet` | 6 | +4 u8 TrophyIndex — remove furniture |

> Trade action codes are NOT in declaration order: **confirm = 2, un-confirm = 3**. 0x34 embeds the resolved
> catalog ItemNo at +8 so the server can cross-check it against the inventory slot.

### 4.3 Chat — gcchat.c (variable-length)
| ID | Builder | Base size | Fields |
|----|---------|-----------|--------|
| 0xB5 | `gcChatStdSend` (70) | 8 + msg | +4 u8 Kind (say/shout/party/ls/emote); +6 char[] msg (NUL-term, ≤0x96). `gcZoneSendQueSet(slot,8,len-1)` |
| 0xB6 | `gcChatNameSend` (143) | 0x16 + msg | +4 u8 flag=0; +5 char[15] target name (<16, auto-uppercased); +0x14 char[] msg (≤0x96). `gcZoneSendQueSet(slot,0x16,len-1)` — **tell** |

Client gating: party chat (Kind 4) requires party ≥ 2; linkshell (Kind 5) requires active comlink; message
clamped to 150 bytes. The 3rd arg to `gcZoneSendQueSet` is the variable-length size adjustment (see §6).

### 4.4 Party / linkshell — gcgroup.c
| ID | Builder | Size | Key fields |
|----|---------|------|-----------|
| 0x6E | `gcGroupSolicitReqSet` | 0xC | +4 u32 UniqueNo; +8 u16 ActIndex; +0xa u8 Kind(0=party,1=ls,2=alliance) — **invite** |
| 0x6F | `gcGroupLeaveSet` | 6 | +4 u8 Kind — leave |
| 0x70 | `gcGroupBreakupSet` | 6 | +4 u8 Kind — disband (leader-only, client checks leader bit) |
| 0x71 | `gcGroupStrikeSet` / `Strike2Set` | 0x1C | +4 u32 UniqueNo; +8 u16 ActIndex; +0xa u8 Kind; +0xc char[15] name — **kick** (by index or by name) |
| 0x73 | `gcGroupChangeSet` | 0xC | +4 u32 UniqueNo; +8 u16 ActIndex; +0xa u8 Kind; +0xb u8 ChangeKind — leader change by index |
| 0x74 | `gcGroupSolicitResSet` | 6 | +4 u8 Res — accept/decline invite |
| 0x76 | `gcGroupListReqSet` | 6 | +4 u8 Kind — member list request |
| 0x77 | `gcGroupChangeSet2` | 0x16 | +4 char[15] name; +0x14 u8 Kind; +0x15 u8 ChangeKind — leader change by name |
| 0x78 | `gcGroupCheckID` | 4 | header-only (params go to zone work, not packet) |
| 0xC3 | `gcGroupComlinkMakeSet` | 6 | +4 u8 State — linkshell create |
| 0xC4 | `gcGroupComlinkActiveSet` | 0x18 | +4/+5 packed RGB nibbles; +6 u8 inv slot; +7 u8 ActiveFlg; +8 char[15] LS name — equip/activate pearl |
| 0xE1 | `gcGroupGetComlinkMessage` | 0x8C | +4 u8 flags; +6 u16 seq; +8 u32 UniqueNo |
| 0xE2 | `gcGroupSetComlinkMessage`(+4 hi=0x40) / `…PubMessage`(0x80) / `…MessageAccessRight`(level bits) | 0x8C | +4 op-nibble; +5 access levels; +6 u16 seq; +0xc msg[128] |
| 0xE3 | `gcGroupGetComlinkPubMessage` | 0x8C | +4 flags; +6 seq; +8 UniqueNo |
| 0xE4 | `gcGroupGetComlinkMessageAccessRight` | 0x8C | +4 =0xb0; +6 seq |

> 0xE2 multiplexes three operations via the high nibble of byte +4 (0x40 set private msg, 0x80 set public
> msg, level-bit pattern for set-access). 0xE1/E3/E4 share the 140-byte frame with a 16-bit seq at +6.

### 4.5 Shops (NPC vendors) — gcshop.c
| ID | Builder | Size | Fields |
|----|---------|------|--------|
| 0x82 | `gcShopReqSet` | 8 | +6 u8 ShopItemOffsetIndex — open/page |
| 0x83 | `gcShopBuyReqSet` | 0x10 | +4 u32 ItemNum; +0xa u16 ShopItemIndex; +0xc u8 inv slot — **buy** |
| 0x84 | `gcShopSellReq` | 0xC | +4 u32 ItemNum; +8 u16 catalog item-ID (from inv); +0xa u8 inv slot — **sell** |
| 0x85 | `gcShopSellSet` | 6 | +4 u16 = 1 — confirm sell |

### 4.6 Bazaar (player vending) — gcbazaar.c
| ID | Builder | Size | Fields |
|----|---------|------|--------|
| 0x104 | `gcBazaarExitSet` | 4 | header-only — leave a bazaar |
| 0x105 | `gcBazaarListReq` | 0xC | +4 u32 UniqueNo; +8 u16 ActIndex — request a player's bazaar list |
| 0x106 | `gcBazaarBuyReq` | 0xC | +4 u8 BazaarItemIndex; +8 u32 BuyNum — **buy from bazaar** |
| 0x109 | `gcBazaarOpenSet` | 4 | header-only — open own bazaar |
| 0x10A | `gcBazaarItemSet` | 0xC | +4 u8 inv slot; +8 u32 Price — set item price |
| 0x10B | `gcBazaarCloseSet` | 8 | +4 u32 AllListClearFlg — close bazaar |

Tax math is client-side too: `gpBazaarGetTax` / `gpBazaarGetPriceIncludingTax`
(`src/main/net/game_prot/gpbazaar.c`) — informational; the server is authoritative on gil.

### 4.7 Auction House — gcauction.c (ALL via packet 0x4E)
**Every AH operation is packet 0x4E, 0x3C (60) bytes**, subtype byte at **+4**, box-slot at **+5**. Common
frame: +4 subtype, +5 aucWorkIndex (0-7), +6/+7 = 0, +8..+0x13 zeroed-then-filled, +0x14..+0x3b zeroed
(item-name region used by the reply).

| Subtype @+4 | Builder | Extra fields |
|-------------|---------|--------------|
| 0x01 | `SendAuction_Transfer` | +8 u16 0x4254("BT"); +0x10 u16 1; +0x12 u16 1 — init handshake |
| (arg) | `SendAuction_MenuControl` | +4 = command byte |
| 0x04 | `SendAuction_AskCommit` | +8 u32 commission; +0xc u16 itemWorkIndex; +0xe u16 itemNo; +0x10 u32 stacks |
| 0x05 | `SendAuction_Info` | — |
| 0x0A | `SendAuction_work` | +5 box index (0xff = all) |
| 0x0B | `SendAuction_LotIn` | +5 box; +8 u32 limitPrice(1..1e9); +0xc u16 itemWorkIndex(0..0x50); +0x10 u32 stacks |
| 0x0C | `SendAuction_LotCancel` | +5 box(0-7) |
| 0x0D | `SendAuction_LotCheck` | +5 box(0-7) |
| 0x0E | `SendAuction_Bid` | +5 box; +8 u32 bidPrice(1..1e9); +0xc u16 itemNo(1..0xffff); +0x10 u32 stacks(0..99) |
| 0x0F | `SendAuction_Get` | +5 box(0-7) |
| 0x10 | `SendAuction_Clear` | +5 box(0-7) |

Client validates ranges (box 0-7, price 1..999,999,999, stacks 0..99, itemNo ≤0xffff) before sending —
**but server must re-validate** (gil balance, item ownership, sale state).

### 4.8 Other gc* builders (catalog)
`gcpost.c` 0x4D (delivery box — all ops multiplexed), `gcconf.c` 0xDC (config/menu), `gcpreference.c`
0x8C/0x8D, `gcswitch.c` 0xA0/0xA1/0xA2 (NPC/event triggers), `gcguild.c` 0xAA-0xAD (guild shop buy/sell),
`gctracking.c` 0xF4/0xF5/0xF6 (widescan/tracking), `gcmap.c` 0xD2 (map marker), `gcblklist.c` 0x3C/0x3D
(blacklist), `gccombine.c` 0x96 (synthesis/crafting), `gcmyroom.c` 0xC9/0xCA/0xFA-0x100 (mog house),
`gcgm.c` 0x1E/0x1F (GM commands). Pattern is identical (`gcZoneSendQueSearch(ID,…)` → fields → `gcZoneSendQueSet`).

---

## 5. Send pipeline — assembly & transmission (`SendProc`)
`src/main/net/game_cli/gczone.c:1734` (DWARF `SendProc__FP7GC_ZONE`, rva 0x000833c0). Per network tick:
1. RTT/throttle bookkeeping on `pZone[4]` vs `_pGcMainSys+0x118` (gczone.c:1750-1762).
2. Acquire a UDP send buffer `ntUdpSendGet` (gczone.c:1773); write IP/port header (+0xc IP, +8 const 2,
   +0xa port) at gczone.c:1782-1784.
3. `gpGamePacketSet` → concatenate queued sub-packets into the 0x1c-headed cleartext (gczone.c:1785).
4. `enAcvSet` → compress+MD5+blowfish into the UDP payload at `+0x40` (gczone.c:1790/1797); store size at +4.
5. `ntUdpSendLose` transmits; bump outgoing seq (gczone.c:1811-1814); periodically `enAcvChange` rotates key.

**Server-relevance:** the UDP datagram = `[transport header][0x1c cleartext game header][blowfish(huffman(payload)+len+MD5)]`.

---

## 6. Rate limiting, throttling & queueing (exploit-relevant)

### 6.1 Per-packet-ID send throttle — `gcZoneSendQueSearch`
`src/main/net/game_cli/gczone.c:1097`. When `SearchSub == 0`, the function enforces a **per-command
minimum interval**:
```c
// gczone.c:1115-1142
last  = *(int*)(zone + Command*4 + 0x25208);   // last-send timestamp for this packet ID
inter = *(int*)(zone + Command*4 + 0x25638);   // min interval for this packet ID
if (last && ntTimeGet() < last + inter) {        // too soon →
    if (pGame->0x448 < 0x65) return 0;            // drop the send (flood guard counter)
}
*(int*)(zone + Command*4 + 0x25208) = ntTimeGet(); // stamp
```
- There is a **per-packet-ID interval table at zone+0x25638** and a **last-send-time table at zone+0x25208**
  (each indexed `Command*4`, command < 0x10c). A `+0x448` counter on `_pGame` acts as a flood budget.
- **Bypass:** when `SearchSub != 0` the throttle block is skipped (the call goes straight to
  `enQueSearch2`). Notably the **action packet 0x1A uses `SearchSub=0`** in `makeActionPacket`
  (commandcalc.cpp:1703) so it IS throttled by the 0x1A interval; but several command-line action emitters
  call `gcZoneSendQueSearch(0x1a)` with no sub-args (default 0) — still throttled. Trade/equip/etc. that pass
  a non-zero SearchSub (a dedup key) bypass the time throttle but instead **coalesce** into one slot.
- **Server-relevance:** the client self-throttles, but an emulator MUST NOT rely on it. The interval table is
  client-side and trivially patchable; enforce recast/cooldown/action-rate authoritatively.

### 6.2 Coalescing / dedup — `enQueSearch2`
`src/main/net/engine_net/enque.c:39`. The `SearchSub`/`SetSub` args are a dedup key: if a pending slot with
the same `(ID, SearchSub)` exists it is **reused/overwritten** (enque.c:66), so e.g. repeated equip changes to
the same slot collapse to the last value. Ring buffer of 0x110-byte slots; `SpecialNum` reserves slots for
priority commands (`enQueSpecialSet`, enque.c:185). `enQueFreeNum` (enque.c:212) reports free slots; when the
queue is full sends are dropped and `_pDebug+0xa4` (drop counter) increments (enque.c:107).

### 6.3 Size encoding — `enQueSet` / `enQueAddSizeGet`
`enQueSet` (enque.c:121) stores size as **32-bit words**: `byte[+1] = ((size+addsize+3)>>2) & 0x7f) << 1`
(low bit reserved). Max packet body **0x104 bytes** (enque.c:138 bounds check). `enQueAddSizeGet`
(enque.c:169) = `PacketSize*4 - StructSize`, used for variable-length string packets (the 3rd
`gcZoneSendQueSet` arg). So **max single sub-packet = 0x104 (260) bytes**, word-aligned.

---

## 7. Anti-tamper / integrity summary (what the server can/can't trust)

- **Integrity:** every packet carries an MD5 over its (compressed) payload (§1.2) and is blowfish-encrypted
  with a per-session, rotating 16-byte key (§1.3/1.5). A forged packet without the right key + valid MD5 is
  rejected by `enAcvGet`. **An emulator must possess/derive the session key to decrypt at all.**
- **Login binding:** zone-in carries `MD5(accountName || key)` as a login ticket (§1.6, §3 packet 0x0A/0x10)
  plus a client version, region, and language — the server should validate these.
- **No meaningful client-side game-rule validation.** The 0x1A action packet clamps only spell id `0..0x1ff`;
  range/level/recast/distance/LoS/MP are NOT checked client-side. Auction/trade builders do range-clamp
  numeric fields but the server must independently validate gil, ownership, and state.
- **Self-throttle is advisory** (§6.1) — patchable; the server must rate-limit authoritatively.
- **Position** is reported as `int(coord*1000)` + 1-byte direction (§3); the server's distance checks must
  decode dir via `dir_cli = dir_net*2π/256` and treat coords as floats. Position rides the standard queued
  sub-packet stream (no dedicated 0x15 builder in this build).

---

## Appendix — packet ID → builder quick index
0x0A/0x10 zone-in/login (StepCalc) · 0x0B logout/zonechange · 0x0D zone-confirm · 0x0F client status ·
0x17 char-data req · **0x1A action (commandcalc makeActionPacket)** · 0x1E/0x1F GM · 0x28-0x2A item drop/move/attr ·
0x32-0x34 trade · 0x37 use item · 0x38 debug-make · 0x39 list req · 0x3A sort · 0x3C/0x3D blacklist ·
0x41/0x42 furniture · 0x4D delivery box · 0x4E auction (all subtypes) · 0x50 equip · 0x6E-0x78 party ·
0x82-0x85 shop · 0x8C/0x8D preference · 0x96 synthesis · 0xA0-0xA2 switch/event · 0xAA-0xAD guild ·
0xB5/0xB6 chat/tell · 0xC3/0xC4 linkshell · 0xC9/0xCA + 0xFA-0x100 mog house · 0xD2 map · 0xDC config ·
0xDD-0xE0 inspect/usermsg · 0xE1-0xE4 comlink msg · 0xF0 rescue · 0xF2 submap · 0xF4-0xF6 tracking/widescan.

*Builders confirmed present via grep of `gcZoneSendQueSearch(` across `src/main/net/game_cli/` and
`src/main/actor/commandcalc.cpp`. DWARF symbol names quoted from `// symbol:` headers; field names inferred.*
