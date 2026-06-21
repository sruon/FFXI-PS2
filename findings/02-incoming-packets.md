# 02 — Incoming (Server → Client) Packet Handling

Scope: how the FFXI **PS2** client (`SCUS_972.66`, 2003) parses zone/game-server
packets and what gameplay semantics each field carries. All file/line references
are into the decompiled tree; addresses are `image_base + rva` (image_base
`0x00100000`). Names tagged **[DWARF]** come from recovered debug symbols (struct
type names like `GP_SERV_*`, handler names like `RecvLogIn`); names tagged
**[inferred]** are my reading of raw offset accesses (the structs themselves are
**not** present as headers — handlers access fields as `*(T*)(param_3 + off)`).

> ⚠️ This is the **2003 PS2 protocol**, which is *older and different* from the
> retail PC protocol most LSB documentation describes. Packet IDs here are
> **9-bit** and the ID→meaning mapping does **not** match modern retail
> (there is no 0x28 "action", no 0x29 "basic message" in this build). Treat the
> retail/XiPackets cross-reference as approximate. The DWARF `GP_SERV_*` type
> names are authoritative for *this* client.

---

## 1. Transport & packet framing (highest-value)

### 1.1 UDP datagram → fragment loop
`RecvProc(GC_ZONE*)` — `src/main/net/game_cli/gczone.c:1532` (rva 0x00082970) **[DWARF]**

This is the single entry point that turns a received UDP datagram into individual
packet dispatches.

- `ntUdpRecvGet()` returns the raw datagram (`gczone.c:1567`).
- The datagram is **Huffman/ACV-decoded** into `auStack_2770` via
  `enAcvGet(...)` (`gczone.c:1573`). The decoder is shared with the send path
  (`enAcvSet`, table `_pGcMainSys + 0x15c7c`).
- **Outer game header** (decoded buffer `auStack_2770`, struct
  `GP_GAME_PACKET_HEAD` **[DWARF]**, built by `gpGamePacketSet`,
  `src/main/net/game_prot/gpgame.c:17`):
  | Off | Type | Meaning |
  |---|---|---|
  | 0x00 | u16 | `MyCnt`  — sender's outgoing sequence counter (server's count) |
  | 0x02 | u16 | `YouCnt` — last sequence the sender acknowledges from us |
  | 0x04 | u32 | timestamp **ms** (`ntTimeNowGetMSec`) |
  | 0x08 | u32 | timestamp **sec** (`ntTimeNowGetSec`) |
  | 0x1c.. | — | concatenated sub-packets begin here (`pSend + 0xe` words) |
- **Sequence dedup**: incoming `MyCnt` (`auStack_2770[0]`) is compared against
  the last processed counter `*(u16*)(pZone+2)`; out-of-order/duplicate
  datagrams are rejected with the `(u16)(last - new) < 0x8000` window test
  (`gczone.c:1595-1597`). Per-fragment the same window test is applied on the
  fragment's own counter `puVar9[1]` (`gczone.c:1609`).
- After processing, `*(u16*)(pZone+2) = MyCnt` and the ack byte at
  `(int)pZone+0xa` are stored, and `enAcvChange` rotates the decode context
  (`gczone.c:1671-1673`).

**Server relevance:** the server MUST send this 0x1c-byte game header with
correct `MyCnt`/`YouCnt` and the ms/sec timestamps, and the payload must be
Huffman-encoded with the same table the client uses. The dedup window means
replays or counter resets desync the stream. (LSB's xi packet layer already
does sequence/ack; the PS2 timestamp fields and Huffman framing are the
PS2-specific concern if ever targeting this client.)

### 1.2 Per-fragment sub-header & dispatch
Inner loop at `gczone.c:1599-1670`. Each sub-packet starts with a **u16**:

| Bits | Field | Meaning |
|---|---|---|
| `& 0x1ff` | **packet ID** (9 bits) | index into dispatch tables; max 0x1FF |
| bits 9-15 (`*(byte*)(p+1) >> 1`, i.e. `(word>>9)&0x7f`) | **size in 4-byte words** | total fragment length = `words * 4`; a size of 0 is an error (`gczone.c:1601`) |

The second u16 (`puVar9[1]`) is that fragment's own sequence counter (used for
the per-fragment dedup window, `gczone.c:1609`).

Dispatch (`gczone.c:1611-1657`):
```
id = *puVar9 & 0x1ff;
// primary table
if (_pZoneSys[0x29758 + id*4]) handler(pZone, datagramHead, fragptr);
// secondary table (only if id < 0x110 and certain zone-state flags clear)
if (id < 0x110 && _pZoneSys[0x29318 + id*4]) handler2(...);
```
Each handler is called as `handler(GC_ZONE* pZone, GP_GAME_PACKET_HEAD* head,
GP_SERV_xxx* body)` — i.e. **param_3 (3rd arg) points at the fragment's first
u16**, so field offset `+4` in handlers is the first byte *after* the
id/size/seq header.

- Max fragment payload guard: `0x104` words (`gpgame.c:46`, send side).
- Aggregate datagram caps in send builder: stops at ~3999 bytes / queue full
  (`gpgame.c:58-59`).

**Server relevance:** ID is **9 bits**, size is **in 4-byte words** (round all
payloads up to a multiple of 4). The size field is `(word >> 9) & 0x7f`, max
0x7f words = 508 bytes per fragment. ID range 0–0x1FF, but handlers are only
registered/validated for IDs < 0x110.

### 1.3 Two handler tables
Registration helpers (`gczone.c`):
- `gcZoneRecvCallBack2(id, fn)` → **primary** table `_pZoneSys + id*4 + 0x29758`
  (`gczone.c:1067`, rva 0x00081d50). Validates `id < 0x110`.
- `gcZoneRecvCallBack(id, fn)` → **secondary** table `_pZoneSys + id*4 + 0x29318`
  (`gczone.c:1037`, rva 0x00081c80). Validates `id < 0x110`. The secondary
  handler only fires when a set of zone-state flags at `_pGlobalNowZone+0x2513c`
  (mask 0x200/0x400/0x2000/0x4000) are clear (`gczone.c:1622-1654`) — these
  appear to gate UI/sub-window handlers during cutscenes/transitions.

**Server relevance:** the same packet ID can have both a "core" handler (e.g.
update game state) and a "UI" handler (blacklist/myroom/map windows). Both run.

---

## 2. Dispatch map: packet ID → handler → purpose

All 80 registrations resolved (RVA→symbol). `T` = primary table (`gcZoneRecvCallBack2`),
`U` = secondary/UI table (`gcZoneRecvCallBack`). Struct type names are **[DWARF]**.

| ID | Dec | Handler | Struct (GP_SERV_…) | Tbl | Source | Purpose |
|----|-----|---------|--------------------|-----|--------|---------|
| 0x05 | 5  | RecvPacketControl | PACKETCONTROL | T | gczone.c | Sets server tick/latency budget (`_pGcMainSys+0x118`) |
| 0x06 | 6  | RecvNaraku | NARAKU | T | gcgm.c | GM "naraku"/jail teleport control |
| 0x08 | 8  | RecvEnterZone | ENTERZONE | T | gczone.c | 0x20-byte zone-enter blob → `pZone+0x251e7` |
| 0x09 | 9  | RecvMessage | MESSAGE | T | gczone.c | Server text/debug-command message channel |
| 0x0A | 10 | RecvLogIn | LOGIN | T | gczone.c | **Zone-in / player spawn** (pos, look, flags, time) |
| 0x0B | 11 | RecvLogOut | LOGOUT | T | gczone.c | Logout / zone-change handshake |
| 0x0D | 13 | RecvDebugPc | CHAR_PC | T | gczone.c | **PC actor existence/visibility sync** |
| 0x0E | 14 | RecvDebugNpc | CHAR_NPC | T | gczone.c | **NPC/mob actor existence/visibility sync** |
| 0x12 | 18 | RecvGm | GM | T | gcgm.c | GM data |
| 0x13 | 19 | RecvGmCommand | GMCOMMAND | T | gcgm.c | GM command |
| 0x17 | 23 | RecvStdChat | CHAT_STD | T | gcchat.c | **Standard chat** (say/shout/tell/party/LS/…) |
| 0x1C | 28 | RecvItemMax | ITEM_MAX | T | gcitem.c | Container capacities (3 bytes) |
| 0x1D | 29 | RecvItemSame | ITEM_SAME | T | gcitem.c | "inventory unchanged"/finalize flag |
| 0x1E | 30 | RecvItemNum | ITEM_NUM | T | gcitem.c | Item stack count update for a slot |
| 0x1F | 31 | RecvItemList | ITEM_LIST | T | gcitem.c | **Inventory slot item id+count** |
| 0x20 | 32 | RecvItemAttr | ITEM_ATTR | T | gcitem.c | Item slot extended attributes (0x18-byte blob) |
| 0x21 | 33 | RecvItemTradeReq | ITEM_TRADE_REQ | T | gcitem.c | Trade request |
| 0x22 | 34 | RecvItemTradeRes | ITEM_TRADE_RES | T | gcitem.c | Trade response |
| 0x23 | 35 | RecvItemTradeList | ITEM_TRADE_LIST | T | gcitem.c | Trade window (their offer) |
| 0x24 | 36 | RecvItemPresent | ITEM_PRESENT | T | gcitem.c | Gift/present item |
| 0x25 | 37 | RecvItemTradeMyList | ITEM_TRADE_MYLIST | T | gcitem.c | Trade window (my offer) |
| 0x2B | 43 | RecvChannelItem | CHANNEL_ITEM | T | gcitem.c | Channel/quest item |
| 0x2C | 44 | RecvChannelState | CHANNEL_STATE | T | gcitem.c | Channel state |
| 0x3C | 60 | RecvShopList | SHOP_LIST | T | gcshop.c | NPC shop inventory |
| 0x3D | 61 | RecvShopSell | SHOP_SELL | T | gcshop.c | Shop sell result |
| 0x3E | 62 | RecvShopOpen | SHOP_OPEN | T | gcshop.c | Open shop |
| 0x3F | 63 | RecvShopBuy | SHOP_BUY | T | gcshop.c | Shop buy result |
| 0x41 | 65 | gcRecvBlackList | — | U | gcblklist.c | Blacklist contents |
| 0x42 | 66 | gcRecvBlackEdit | — | U | gcblklist.c | Blacklist add/remove |
| 0x4F | 79 | RecvEquipClear | EQUIP_CLEAR | T | gcequip.c | Clear an equipment slot |
| 0x50 | 80 | RecvEquipList | EQUIP_LIST | T | gcequip.c | **Equip slot → inventory index map** |
| 0x64 | 100| receivePreferenceData | PREFERENCE_DATA | U | gcpreference.c | Server-pushed preference/config blob |
| 0x6F | 111| gcRecvCombine | — | T | gccombine.c | Synthesis/combine result |
| 0x70 | 112| gcRecvCombineInfo | — | T | gccombine.c | Synthesis info |
| 0x78 | 120| RecvStart (switch) | SWITCH_START | T | gcswitch.c | "Switch"/event-trigger start |
| 0x79 | 121| RecvProc (switch) | SWITCH_PROC | T | gcswitch.c | "Switch"/event-trigger proc |
| 0x82 | 130| gcRecvGuildBuy | — | T | gcguild.c | Guild shop buy |
| 0x83 | 131| gcRecvGuildBuyList | — | T | gcguild.c | Guild shop buy list |
| 0x84 | 132| gcRecvGuildSell | — | T | gcguild.c | Guild shop sell |
| 0x85 | 133| gcRecvGuildSellList | — | T | gcguild.c | Guild shop sell list |
| 0x86 | 134| gcRecvGuildOpen | — | T | gcguild.c | Open guild shop |
| 0x96 | 150| gcRecvMyroomEnter | — | U | gcmyroom.c | Mog House enter |
| 0x97 | 151| gcRecvMyroomExit | — | U | gcmyroom.c | Mog House exit |
| 0x98 | 152| gcRecvMyroomIs | — | U | gcmyroom.c | Mog House query |
| 0x99 | 153| gcRecvMyroomExist | — | U | gcmyroom.c | Mog House exists |
| 0x9A | 154| gcRecvMyroomPlant | — | U | gcmyroom.c | Garden plant |
| 0x9B | 155| gcRecvMyroomRaise | — | U | gcmyroom.c | Garden tend |
| 0x9C | 156| gcRecvMyroomHarvest | — | U | gcmyroom.c | Garden harvest |
| 0x9D | 157| gcRecvMyroomDiary | — | U | gcmyroom.c | Garden diary |
| 0x9E | 158| gcRecvMyroomJob | — | U | gcmyroom.c | Mog House job-related |
| 0xA0 | 160| gcRecvMapGroup | — | U | gcmap.c | Region/map group data |
| 0xAA | 170| RecvMagicData | MAGIC_DATA | T | gcmagic.c | Known-spells bitmap |
| 0xAB | 171| RecvFeatData | FEAT_DATA | T | gcfeat.c | Abilities/merits "feat" data |
| 0xAC | 172| RecvCommandData | COMMAND_DATA | T | gcfeat.c | Usable command data |
| 0xB4 | 180| RecvConf | CONFIG | T | gcconf.c | **Char config flags (0xC bytes)** |
| 0xB5 | 181| FDtRecvGmParam | FAQ_GMPARAM | T | fdtFFXiService.cpp | Bug-DB/FAQ GM params |
| 0xB6 | 182| FDtRecvGmNotice | SET_GMMSG | T | fdtFFXiService.cpp | GM broadcast notice |
| 0xC8 | 200| RecvGroupTbl | GROUP_TBL | T | gcgroup.c | **Party/alliance roster table** |
| 0xC9 | 201| RecvEquipInspect | EQUIP_INSPECT | T | gcequip.c | /check equipment |
| 0xCA | 202| RecvInspectMessage | INSPECT_MESSAGE | T | gcequip.c | /check bazaar/comment text |
| 0xCC | 204| RecvComlinkMessage | LINKSHELL_MESSAGE | T | gcgroup.c | Linkshell message |
| 0xD2 | 210| RecvTrophyList | TROPHY_LIST | T | gcitem.c | Trophy/key-item list |
| 0xD3 | 211| RecvTrophySolution | TROPHY_SOLUTION | T | gcitem.c | Trophy/key-item detail |
| 0xDC | 220| RecvGroupSolicitReq | GROUP_SOLICIT_REQ | T | gcgroup.c | Party invite request |
| 0xDD | 221| RecvGroupList | GROUP_LIST | T | gcgroup.c | **Party member status row** |
| 0xDE | 222| RecvGroupSolicitNo | GROUP_SOLICIT_NO | T | gcgroup.c | Party invite decline |
| 0xDF | 223| RecvGroupAttr | GROUP_ATTR | T | gcgroup.c | Party attributes |
| 0xE0 | 224| RecvComlink | GROUP_COMLINK | T | gcgroup.c | Linkshell link state |
| 0xE1 | 225| RecvCheckID | GROUP_CHECKID | T | gcgroup.c | Party member id check |
| 0xE2 | 226| RecvGroupList2 | GROUP_LIST2 | T | gcgroup.c | Party member status (variant 2) |
| 0xF4 | 244| RecvList (tracking) | TRACKING_LIST | T | gctracking.c | Wide-scan list |
| 0xF5 | 245| RecvPos (tracking) | TRACKING_POS | T | gctracking.c | Wide-scan position |
| 0xF6 | 246| RecvState (tracking) | TRACKING_STATE | T | gctracking.c | Wide-scan state |
| 0xFA | 250| RecvOperation (myroom) | MYROOM_OPERATION | U | gcmyroom.c | Mog House operation result |
| 0x105| 261| RecvList (bazaar) | BAZAAR_LIST | T | gcbazaar.c | Bazaar item list |
| 0x106| 262| RecvBuy (bazaar) | BAZAAR_BUY | T | gcbazaar.c | Bazaar buy result |
| 0x107| 263| RecvClose (bazaar) | BAZAAR_CLOSE | T | gcbazaar.c | Bazaar close |
| 0x108| 264| RecvShopping (bazaar) | BAZAAR_SHOPPING | T | gcbazaar.c | Bazaar browsing |
| 0x109| 265| RecvSell (bazaar) | BAZAAR_SELL | T | gcbazaar.c | Bazaar sell |
| 0x10A| 266| RecvSale (bazaar) | BAZAAR_SALE | T | gcbazaar.c | Bazaar sale notice |

Registration callsites are listed per file, e.g. `gcitem.c:24-50`, `gcgroup.c:23-39`,
`gcbazaar.c:21-31`, `gczone.c:69-75` (the core handlers 5,8,9,10,11,13,14).

> **Note:** Auction handlers (`gcauction.c`), mail/post (`gcpost.c`), friend list
> (`gcfriend.c`), login (`gclogin.cpp`), and the cache (`gccache.cpp`) do **not**
> register into the zone dispatch table — auction/post/friend traffic is carried
> over the separate **POL message** subsystem (`gcpolmessage.cpp`, ~298 KB,
> class `FFPMsg_Ctrl`, header type `sqPolMessageHeader`), see §6.

---

## 3. High-value packet structs (field-level)

Offsets below are relative to `param_3` (the fragment pointer). `+0` = the
id/size u16, `+2` = fragment sequence u16, **`+4` = first payload byte.**

### 3.1 0x0A `GP_SERV_LOGIN` — zone-in / local player spawn
`RecvLogIn` — `gczone.c:1926` (rva 0x00083ae0, 5076 bytes — the largest handler).
`param_3` here is typed `ushort* puStack_10`, so indices below are in **u16 units**
unless byte-addressed.

| Offset (bytes) | Field (inferred) | Notes |
|---|---|---|
| +0x04 | u32 UniqueNo (server actor id) | stored `pZone+0x25170`; compared everywhere as "self id" |
| +0x06 | f32 X | position; copied to `pZone+0x25140` and backup `+0x25150` |
| +0x08 | f32 Y | `pZone+0x25144` |
| +0x0A | f32 Z | `pZone+0x25148` |
| +0x0B | u8 dir | heading; passed through `enDirNetToCli()` → `pZone+0x2515c` |
| +0x0C | u16 zone-ish | `& 0x1fff` → `pZone+0x251e4` (zone/area id, 13-bit) |
| +0x10 | u16 packed look/flags | race(low bits), face, etc. — many single-bit extracts into `pZone+0x29295/0x29296` |
| +0x11 | u16 packed flags 2 | bits → `pZone+0x29298/0x29299` |
| +0x18 | u32 | `& 0xffff` used as login-message id (`XiStrGet(5, val-100)`); stored `pZone+0x25138` |
| +0x1a/+0x1c/+0x1e | times | `ntTimeSet`/`ntGameTimeInit` (Vana'diel + server clock sync) |
| +0x40 | u8/u32 LoginType | `1`=normal,`3`,`4`,`5` set different zone flags; first byte also written to XiInfo (`gczone.c:1945,1951`) |
| +0x42 | char[0x10] name | local player name → `_pGcMainSys+0x68`, `pZone+0x25184` |
| +0x52 | u32 | `pZone+0x292bc` |
| +0x21,+0x31..+0x35,+0x3a,+0x3b,+0x36,+0x38,+0x55 | misc u16/u32 | model/main-job/level/etc. into `pZone+0x29268..0x292aa` |

**Server relevance:** the canonical zone-in. Position is 3×f32, heading is a u8
needing the net→client direction transform. Zone id is **13 bits** of +0x0C.
`LoginType` at +0x40 drives whether the client treats this as a fresh login vs
a zone transfer (sets flags 0x1/0x100/0x200/0x2000). The time fields seed both
the network clock and the Vana'diel game clock — wrong values desync day/night.

### 3.2 0x0D `GP_SERV_CHAR_PC` / 0x0E `GP_SERV_CHAR_NPC` — actor presence sync
`RecvDebugPc` — `gczone.c:2379` (rva 0x00085490); `RecvDebugNpc` — `gczone.c:2471`
(rva 0x00085930). These maintain the **actor table** that decides which entities
the client believes exist near it.

| Offset | Field (inferred) | Notes |
|---|---|---|
| +0x04 | u32 UniqueNo | server entity id |
| +0x08 | u16 ActIndex | **local actor slot**, range-checked `< 0x700` (1792 slots); used as `pZone + ActIndex*8 + 0x25a68` |
| +0x0A | u8 flag bitfield | request/visibility bits; `0x20` = "remove/clear actor" (sets slot id to 0xffffffff), low bits OR'd into `+0x25a6c` |
| (NPC +0x30) | u8 type | `& 7 == 1` selects PC-like vs NPC handling branch |

Logic: per actor slot it stores `{UniqueNo @ +0x25a68, flags u16 @ +0x25a6c}`
(stride 8). If the server reports a different UniqueNo for an existing slot, the
client sends a **0x17 DebugReqChar** request (`gcZoneSendQueSearch(0x17)`,
`gczone.c:2401`, struct built in `DebugReqCharSend` `gczone.c:2344`) to ask the
server for the full actor data. The `0x1f`/`0x17`/`0x07` magic flag values gate
whether a re-request is sent.

**Server relevance:** this is the PS2 client's **actor visibility protocol** —
a compact "does actor N (UniqueNo U) exist?" sync. ActIndex must stay `< 0x700`.
Bit `0x20` of the flag byte means despawn. The client will *request* missing
actor data via outgoing 0x17, so the server must answer those. This is the
mechanism that replaces retail's continuous 0x0D/0x0E full updates.

### 3.3 0x17 `GP_SERV_CHAT_STD` — chat
`RecvStdChat` — `gcchat.c` (rva 0x0019e190).

| Offset | Field (inferred) | Notes |
|---|---|---|
| +0x01 | u8 (size hi) | part of fragment header; used to compute body length (`enQueAddSizeGet`) |
| +0x04 | u8 flags | bit `0x01` = system/auto message (skips blacklist check) |
| +0x05 | char[0x0F] sender name | `gcCheckBlack(name)` filters blacklisted senders |
| +0x14 | u8 **chat mode** | switch, see enum below |
| +0x15 | char[] message body | length capped at 0x96 (150) bytes |

**Chat-mode enum** (from the `switch` at `gcchat.c`):
`0`=Say, `1`=Shout, `3`=Tell (sets LastTeller), `4`=Party, `5`=Linkshell,
`6`/`7`=emote-style (name+body only), `8`=?, `9`/`a`/`b`=ignored,
`0xC`=GM tell (opens GM tell window), `0xD`/`0xE`/`0xF`/`0x10`=server/system
broadcasts (body only), `0x11`/`0x12`/`0x13`/`0x14`=more system, `0x15`=GM tell
variant (sets LastTeller). (Exact labels beyond 0–5 are **[inferred]**.)

**Server relevance:** sender name field is **15 bytes**; chat mode is at +0x14;
body starts at +0x15 and is length-limited to 150 bytes client-side. Flag bit
0x01 bypasses the local blacklist. Modes 0xC/0x15 trigger GM-tell UI.

### 3.4 Inventory family (0x1C/0x1E/0x1F/0x20/0x1D)
All write into the inventory model rooted at `_pGlobalNowZone`, addressed as
`base + container*0xCA8 + slot*0x28` (container stride **0xCA8**, slot stride
**0x28** = 40 bytes).

- **0x1F `ITEM_LIST`** (`gcitem.c`): `+0xA`=container, `+0xB`=slot,
  `+0x08`=u16 item id (→slot+0x5748), `+0x04`=u32 count (→slot+0x574c),
  `+0x0C`=u8 flags (→slot+0x5750).
- **0x1E `ITEM_NUM`**: `+0x08`=container, `+0x09`=slot, `+0x04`=u32 count,
  `+0x0A`=u8 flags. Count 0 ⇒ `memset` the 0x28-byte slot (item removed).
- **0x20 `ITEM_ATTR`**: `+0x0E`=container, `+0x0F`=slot, `+0x0C`=u16 id,
  `+0x04`=u32 count, `+0x08`=u32 (→slot+0x5754), `+0x10`=u8 flags,
  `+0x11`=**0x18-byte extdata blob** (→slot+0x5758). This is the augment/signature data.
- **0x1C `ITEM_MAX`**: 3-byte container-capacity array → `pZone+0x81e1`.
- **0x1D `ITEM_SAME`**: `+0x04`==1 ⇒ set "inventory ready" flag 0x800 and bump a
  counter (end-of-inventory-stream marker).

**Server relevance:** slot record is 40 bytes `{u16 id, u8 slot, ?, u32 count,
u8 flags, ..., 0x18 extdata}`. Container indices and slot indices are u8. Send
0x1D with byte 1 to signal "inventory snapshot complete". Sending count 0 in
0x1E deletes the slot.

### 3.5 0x50 `GP_SERV_EQUIP_LIST` / 0x4F `EQUIP_CLEAR`
`RecvEquipList` — `gcequip.c` (rva 0x0019d860):
```
pZone[ equipSlot*8 + 0x81ec ] = inventoryIndex;   // +5 = equip slot, +4 = inv index (u8)
```
i.e. `+0x04` = u8 inventory index, `+0x05` = u8 equipment slot id. Maps an
equipment slot to the inventory slot holding the worn item.

**Server relevance:** equipment is communicated as (equip-slot → inventory-index)
pairs, not item ids directly; the client resolves the item via the inventory
model (§3.4). Equip slot id is a u8.

### 3.6 Party / alliance (0xC8 `GROUP_TBL`, 0xDD `GROUP_LIST`)
Party model: 3 parties (alliance), party index 0–2 with **`2` remapped to `0`**
(`gcgroup.c`); base `_pGlobalNowZone + party*0x5F4 + 0x4490`, member stride
**0x4C**, up to **0x14 (20)** member entries per party array.

- **0xC8 `RecvGroupTbl`**: array of `{u32 UniqueNo @ +N*8, u16 @ +N*8+? }` (stride
  8 in the packet), 20 entries; reconciles the member table — members not present
  in the new table are `memset` to 0 (removed). `param_3[1]` low byte = party index.
- **0xDD `RecvGroupList`**: a single member's status row:
  `+0x04`=u32 UniqueNo, `+0x08`=u32 (HP/zone?), `+0x0C`=u32, `+0x10`=u32,
  `+0x14`=u32 (→member+0x34), `+0x18`=u16 (→member+0x18, likely flags/index),
  `+0x1C`=u8 party index (`2`→0), `+0x1D`/`+0x1E`=u8 (→member+0x2c/0x2d),
  `+0x20`=member name (Huffman string, 0x10 bytes via `enQueStrCpy`).

**Server relevance:** 3-party alliance with party index 0–2 (value 2 collapses
to 0 — likely the "alliance third party" quirk), 20 members per array, 0x4C-byte
member records keyed by UniqueNo. Member name is a Huffman-compressed string.

### 3.7 Other compact structs
- **0x05 `PACKETCONTROL`** (`gczone.c:1853`): `+0x04`=u32 → server tick/latency
  budget written to `_pGcMainSys+0x118` and `pZone+0x10`. Drives the adaptive
  send timer (`SendProc`, `gczone.c:1750-1762`).
- **0x08 `ENTERZONE`** (`gczone.c:1877`): copies a flat **0x20-byte** blob from
  `+0x04` to `pZone+0x251e7`.
- **0x0B `LOGOUT`** (`gczone.c:2203`): `+0x04`=u32 logout code — `1`=zone change,
  `4`=full logout (clears flags, sets flag 1), `8`=error/kick (`XiErrDispose(0xfa1)`),
  default=disconnect. `+0x08`=u32 next-server IP, `+0x0C`=u16 port (`htons`).
  **Server relevance:** this is the zone-transfer/logout handshake — code 1 ⇒
  hand off to the IP/port in +0x08/+0x0C.
- **0xB4 `CONFIG`** (`gcconf.c`): flat **0xC-byte** blob → `pZone+0x19aec`
  (char config flags / game settings).
- **0x09 `MESSAGE`** (`gczone.c:2658`): `+0x04`=u32 sender, `+0x0C`=u8 flags
  (bit 0x10 ⇒ run through blacklist via `gcCheckBlackID`), `+0x0D`=**ASCII text
  body**. The handler parses `key:value` tokens out of the text (debug/command
  message channel), not a structured combat message.

---

## 4. Client-side validation / clamps (server must respect)

- **Actor index** must be `< 0x700` (1792); else error and packet dropped
  (`gczone.c:2392`, `2484`).
- **Packet ID** must be `< 0x110` to register a handler (`gczone.c:1045,1075`);
  IDs ≥ 0x110 are silently undispatched in the secondary table and only the
  primary table is checked up to the array bound.
- **Fragment size word** of 0 is rejected (`gczone.c:1601`); send-side caps a
  fragment at 0x104 words (`gpgame.c:46`).
- **Sequence window**: a fragment/datagram whose counter is "behind" by ≥ 0x8000
  is dropped as a duplicate (`gczone.c:1595,1609`).
- **Chat body** truncated to 150 bytes (`gcchat.c`); **party** member count
  capped at 20, **party index** 2→0 remap.
- **Inventory count 0** ⇒ slot wiped (`gcitem.c` RecvItemNum).
- Datagram with a stale/zero zone id (`*(iVar5+0xc) != _pGlobalNowZone+0x4174`)
  is treated as control-only and skipped past the dispatch loop (`gczone.c:1572`,
  `LAB_00183210`).

---

## 5. Sequence / ack / timestamp & adaptive timing

- Outgoing header set by `gpGamePacketSet` (`gpgame.c:17`): `MyCnt`, `YouCnt`,
  ms timestamp, sec timestamp (see §1.1).
- `SendProc` (`gczone.c:1734`) increments the send counter (`(int)pZone+6`,
  skipping 0), adapts the resend interval `pZone[4]` based on the gap between
  our send counter and the server's acked counter (`gczone.c:1750`), bounded by
  the server-provided budget `_pGcMainSys+0x118` (from 0x05 PACKETCONTROL).
- `RecvProc` updates the Vana'diel clock every 30 datagrams
  (`MyCnt % 0x1e == 0 → gcZoneTimeSet`, `gczone.c:1675`).
- Round-trip/latency tracking: `pZone[0xa4c2..0xa4c4]` accumulate elapsed time
  between sends and acks; if it exceeds a threshold (`pZone+0x29306`) a
  "connection lost"-style flag (`pZone+0xa4c0`) is raised (`gczone.c:1702-1713`,
  `SendProc` `1826-1833`).

---

## 6. POL message subsystem (separate channel — out of zone dispatch)

`gcpolmessage.cpp` (class `FFPMsg_Ctrl`, header `sqPolMessageHeader` **[DWARF]**)
is a **distinct** reliable-message layer used for auction house, mail/delivery
box, friends, and other "POL" services. Entry `gcPolMessageProc()` runs each
frame from `ntGameProc2` (`ntgameproc.cpp:100`). Receive path:
`FFPMsg_Ctrl::StartRecvProc(sqPolMessageHeader*)` — `gcpolmessage.cpp:851`,
dispatching to `FFMsg_MsgOperation` objects keyed by an `opID`
(`gcpolmessage.cpp:902`, `1625`). The `recvMailCount` counter is at
`FFPMsg_Ctrl+...` (`gcpolmessage.cpp:928`).

**Server relevance:** auction/mail/friend packets do **not** use the 9-bit zone
packet IDs in §2; they ride the POL message envelope. If targeting these
features, the `sqPolMessageHeader`/`FFMsg_MsgOperation` opID scheme must be
modeled separately. (Not fully enumerated here — flag if needed as a follow-up;
this file's mandate is the zone game-packet dispatch.)

---

## 7. Open items / uncertainties

- `GP_SERV_*` struct **type names are DWARF-authoritative**, but the individual
  **field offsets/names are inferred** from raw pointer arithmetic (no struct
  headers were emitted). Offsets quoted are reliable; field *semantics* beyond
  the obvious (id/pos/count/name) are my best reading.
- The chat-mode enum labels past mode 5 are inferred from format-string usage,
  not named constants.
- No 0x28 "action" / 0x29 "basic message" packets exist in this build — combat
  results appear to arrive via 0x09 MESSAGE (text channel) and actor state via
  0x0D/0x0E. This is a genuine PS2/2003-era protocol difference vs retail PC.
- The POL message subsystem (auction/mail/friends, §6) was not opcode-mapped.
- `RecvNaraku` (0x06) and several guild/combine handlers were not field-decoded
  (low gameplay priority).
