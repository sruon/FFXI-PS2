# UI Data Systems — Gameplay Limits & Constraints

Source: decompiled FFXI PS2 client SCUS_972.66 (2003), decomp **version 972 (sha256=d4622c1cc4a3012c, image_base=0x00100000)**, Ghidra 12.0.4 + recovered DWARF1 symbols.

Scope: inventory, equipment, auction house, party/alliance, search, trade, bazaar, combine/synthesis, chat, linkshell, tell, friend, blacklist.

**Confidence legend**
- **CERTAIN** = literal constant present in the decompiled code, or a field size from authoritative PS2 DWARF1 (`mdebug`) struct headers.
- **INFERRED** = derived from offset/loop arithmetic, or from undecompiled rodata (table contents not present in source).

**Note on constants:** This decompilation inlines almost all limits as numeric literals (loop bounds, array strides, `memset` sizes) rather than `#define`s. Values below are the raw literals from the code.

---

## 1. Hard numeric limits / caps (lead table)

| Limit | Value | Certainty | Location |
|---|---|---|---|
| Inventory slots per container | **81 entries (0..80), ~80 usable** | CERTAIN | `KmItem.h:23` (`m_ItemNum` 81b); `gcitem.c:54` (`< 0x51`) |
| Item containers (this client) | **3** (Inventory / Mog Safe / Locker) | CERTAIN | `gcitem.c:53` (`< 3`); `ykWndItem.h` GetCate |
| Per-container max size source | **server-provided, 3 bytes** | CERTAIN | `gcitem.c:1093` (`memcpy(...,3)`); `gcitem.c:497` |
| Item slot record stride | **0x28 (40) bytes**; container stride 0xca8 (3240) | CERTAIN | `gcitem.c:153` |
| Max items per stack | **server-provided per item (uint16)**; per-slot stored as **1 byte (≤255)** | CERTAIN | `CTkItemDataMenber.h:23` (`m_pStackMax`); `gcitem.c:1152` |
| Item ID field width | **uint16 (0..65535)** | CERTAIN | `CTkItemInfo.h:22`; `gcitem.c:1150` |
| Item quantity field width | **uint32** | CERTAIN | `gcitem.c:1119` |
| Equipment slots | **16 (0x10), index 0..15** | CERTAIN | `gcequip.c:32` (`< 0x10`); `gcequip.c:633` |
| Max party size | **6** | CERTAIN | `ykWndParty.cpp:837` (`iVar2 = 6`); `:372` |
| Max alliance size | **18 (0x12)** | CERTAIN | `ykWndParty.cpp:835` (`iVar2 = 0x12`); `:909` (`0x11 <`) |
| Parties per alliance | **3** (own + Raid1 + Raid2) | CERTAIN(widgets)/INFERRED(18÷6) | `ykWndParty.cpp:103,129,148` |
| Max auction listings (own sales box) | **8-slot storage; 7 active sells** | CERTAIN | `gcauction.c:21` (`< 8`); sell-scan `ykWndAuction.cpp:1559` (`< 7`) |
| Auction list/bid price range | **1 .. 999,999,999** (int32) | CERTAIN | `gcauction.c:354`, `:560`; `ykWndAuction.cpp:1736` |
| Auction bid quantity (stacks) | **0 .. 99** | CERTAIN | `gcauction.c:563` |
| Auction fee table | **81 entries × int32** (indexed by item category byte) | CERTAIN | `ykWndAuction.cpp:2212` (`0x144` bytes) |
| Trade window slots | **gil (slot 0) + up to 8 item slots** (10-entry backing array) | CERTAIN | `TkTrade.cpp:343,472`; `gcitem.c:678` (`< 10`) |
| Trade gil field | **int32** (no client clamp; server enforces) | CERTAIN(width) | `TkTrade.cpp:467` |
| Bazaar price field | **uint32 per unit** (no client max) | CERTAIN | `gcbazaar.c:251,264` |
| Bazaar list entry stride | **44 bytes**; list region 0xdec (3564) | CERTAIN | `gcbazaar.c:121,327` |
| Combine (synthesis) | **1 crystal + up to 8 ingredients** | CERTAIN | `ykWndCombine.cpp:155` (`7 <`); `:149`; `gccombine.c:161` |
| Max chat message body | **150 bytes (0x96)** all channels | CERTAIN | `gcchat.c:112` |
| Receive chat buffer | **160 bytes (0xa0)** | CERTAIN | `gcchat.c:218` |
| Max blacklist entries (in-game) | **100** (24-byte stride) | CERTAIN | `gcblklist.c:122,138,192,245` |
| Blacklist refresh throttle | **60 s (0x3c)** | CERTAIN | `gcblklist.c:345,405,452,503` |
| Max POL friend list | **300** = 200 normal + 100 "black" | CERTAIN | `gcfriend.c:1558-1562,473,487` |
| Character name field | **15 chars + NUL = 16-byte buffer** | CERTAIN | `gcchat.c:177,220`; member name `0x10` `gcgroup.c:41` |
| Linkshell name (UI input) | **15 chars (0xf)** | CERTAIN | `ykWndComlink.cpp:1367` |
| Linkshell slots | **3 slot menus, 1 active for chat** | CERTAIN(menus)/INFERRED(semantics) | `ykWndComlink.cpp:316-361` |
| Level filter clamp (search) | **1 .. 99** | CERTAIN | `ykWndSearch.cpp:2371,2382` |
| Search result buffer | **513 (0x201)**; auction history 14; 15 lines shown | CERTAIN | `ykWndSearch.cpp:1184,1200,1394` |
| Visual appearance model slots | **8** (race + 7 gear pieces) | CERTAIN | `gpequip.c:36` (`< 9`) |

Server relevance (overview): every cap above is a *client-side mirror* of a server expectation. Client validation is advisory; LSB must re-enforce all of them. The packet opcodes and field offsets given throughout are the most directly reusable artifacts for an emulator.

---

## 2. Inventory & item storage

**81 slots per container, ~80 usable.**
- `include/KmItem.h:23` — DWARF: `unsigned char m_ItemNum; /* 81b */` (the per-category count array is 81 bytes).
- `gcitem.c:54` — init loop `for (bVar2 = 0; bVar2 < 0x51; ...)` (0x51 = 81). Slot 0 is a reserved/header slot; free-space scans begin at index 1, so usable = 80.
- Slot record stride **0x28 (40 bytes)** (`gcitem.c:153`), container block stride **0xca8 (3240 = 81×40)**.
- Server relevance: matches the era's 80-slot inventory. LSB must send the ITEM_MAX packet before listing items or the client shows 0 free space.

**3 containers only (this 2003 client).**
- `gcitem.c:53` — outer loop `uVar3 < 3`. `GetCate` overrides resolve **0 = Inventory**, **1 = Mog Safe / Bank**, **2 = Locker/Closet** (`ykWndItem.h`, `ykWndBank.cpp:1247`).
- Per-container max capacity is **server-driven**: `RecvItemMax` copies **3 bytes** (one max per container) to zone+0x81e1 (`gcitem.c:1093`); read via `gcItemMaxSpaceGet` (`gcitem.c:497`). No hardcoded "80" — structural ceiling is 81.
- Server relevance: an emulator targeting this client must collapse later containers (satchel/sack/wardrobes) — only 3 exist here.

**Field widths.**
- Item ID = **uint16** on the wire/slot (`CTkItemInfo.h:22`; `gcitem.c:1150`). `m_pItemNo` in item-data is `unsigned int*` but slot value is 16-bit → IDs fit 0..65535.
- Quantity = **uint32** (`gcitem.c:1119,1151`); drop/move/trade carry uint32. Per-slot stack-max stored as **1 byte** (`gcitem.c:1152`) so stacks > 255 impossible (FFXI 12/99 fine).
- Stack max = **uint16 per item from server data** (`CTkItemDataMenber.h:23`), not a client constant. No `12`/`99` literal in client.

---

## 3. Equipment — slots, validation, bitfields

**16 equipment slots (index 0..15).**
- `gcequip.c:32` — `for (uVar2 = 0; uVar2 < 0x10; ...)`; 8-byte stride at base 0x81e8; table 0x1b4 (436) bytes. Reconfirmed at `gcequip.c:105,198,331,596`, incoming-slot validation `gcequip.c:633` (`< 0x10`).
- Canonical order (CERTAIN count, **INFERRED names** — name strings live in undecompiled rodata `equippostbl`, looked up by `CTkEquip::GetEquipNoByName`, `TkEquip.cpp:1302`): `0 main, 1 sub, 2 ranged, 3 ammo, 4 head, 5 body, 6 hands, 7 legs, 8 feet, 9 neck, 10 waist, 11 ear1, 12 ear2, 13 ring1, 14 ring2, 15 back`.
- Equip-change request = opcode **0x50**, carries `(slotNo, itemIndex)` (`gcEquipChangeSet`, `gcequip.c:129-147`).

**Item-data struct (authoritative DWARF, `include/CTkItemDataMenber.h`):** exposes all equip constraints as pointers:
- `+0x0c10 m_pProperty` (uint16, flags), `+0x0c14 m_pStackMax`, `+0x0c5c m_pEquipLv`, `+0x0c60 m_pEquipPos` (slot bitmask), `+0x0c64 m_pEquipRace` (race bitmask), `+0x0c68 m_pEquipJob` (job bitmask), `+0x0c6c m_pUseSkill`, `+0x0c80 m_pEquipAC`, `+0x0c84 m_pDamage`, `+0x0c88 m_pAttackDilay`, `+0x0c90 m_pCastTime`.

**Client-side equip validation — `CTkEquip::CanEquipment` (`TkEquip.cpp:975-1003`):**
- **Equippable:** rejects if `m_pEquipPos == 0` (`TkEquip.cpp:983`).
- **Slot fit (bitmask):** item fits slot `pos` iff `(*m_pEquipPos & (1 << (pos & 0x1f))) != 0` (`TkEquip.cpp:1405`). uint16 → bit N = slot N (main=bit0 … back=bit15).
- **Job (bitmask):** `*m_pEquipJob & (1 << (job & 0x1f))` (`TkEquip.cpp:994-995`) — job id used directly as bit position.
- **Race (bitmask):** `*m_pEquipRace & (1 << (race & 0x1f))` (`TkEquip.cpp:996`).
- **Level:** `playerLevel < *m_pEquipLv` ⇒ cannot equip (`TkEquip.cpp:997`).
- **Two-hand:** equipping a 2H weapon (`kind==4`) checks sub-slot occupancy / `IsBothHandSkill` (errors 0x8a/0x97/0x98, `TkEquip.cpp:1373,1390`).
- Server relevance: all advisory — LSB must re-validate. Slot/job/race bitmasks and the `m_pEquipPos` uint16 (bit=slot) map 1:1 to FFXI item_equipment `slot` and job masks. Job bit = `1 << jobId` (direct index) per this client.

**Appearance/model sanitation — `gpGrapCheck` (`gpequip.c:15-56`):** race byte `< 0x20`, model-race nibble 1..0x1f, and **8 visual model slots** (`gpequip.c:36`, `uVar2 < 9`) clamped to `GP_GRAP_KIND_DATA` (kind nibble `& 0xf000`, id `& 0xfff`). Bad model IDs are silently blanked. These 8 visual slots are distinct from the 16 logical equip slots. Server relevance: bad look packets won't crash but render blank.

---

## 4. Item flag bitfields (m_pProperty, uint16)

- **Bit 15 (0x8000) = RARE** — `TkItemInfo.cpp:364`: `if ((*m_pIDM->m_pProperty & 0x8000) != 0)` draws first flag icon. **CERTAIN.**
- **Bit 14 (0x4000) = EX (Exclusive)** — `TkItemInfo.cpp:371`: `& 0x4000` draws second flag icon. **CERTAIN.**
- **Equippable is NOT in m_pProperty** — it is `m_pEquipPos != 0` (`TkEquip.cpp:983`).
- **No-sale / no-auction / no-trade:** no distinct client branch found in scope. The PS2 client visually decodes only RARE and EX; remaining property bits exist but enforcement is server-side. **INFERRED** that only 0x8000/0x4000 are client-acted.
- Server relevance: LSB's item `flags` bit layout differs from this client — do not assume a shared map; only RARE=0x8000 / EX=0x4000 positions are confirmed here. Sale/auction/trade restrictions must be enforced server-side regardless.

---

## 5. Auction house

**Sales box: 8-slot storage, 7 active sells (off-by-one).**
- Storage/validators bound index `< 8`: `gcauction.c:21` (box work), `:283/:348/:425/:489/:554/:631/:695` (all Send* validators use `7 < index`), `:1192` (dump clamps 0..7). Box array = 8 × 0x28 = 0x168 bytes.
- **Discrepancy (likely client bug, server-relevant):** the SELL/list path `PriceYnCallBack` scans only `for (iVar1 = 0; iVar1 < 7; ...)` for a free slot (`ykWndAuction.cpp:1559`), while the BID path scans `< 8` (`ykWndAuction.cpp:1248`). So normal listing caps at 7 active sales (retail value), but storage/packet validators accept slot index 7 (8th slot).
- Server relevance: enforce **7 active sales per character** server-side; a crafted packet could target slot index 7. LSB should not trust the client cap.

**Price / quantity.**
- List price and bid: **1 .. 999,999,999** int32 (`gcauction.c:354,560`; MoneyCtrl max `ykWndAuction.cpp:1736`). The classic 999,999,999 gil ceiling.
- Price is a 4-byte int at packet offset **+8**.
- Bid quantity (stacks): **0 .. 99** (`gcauction.c:563`). The "stacks" field is effectively a 0/1 single-vs-stack flag (`_g_count = (stackmax == 1)`, `ykWndAuction.cpp:1734`).
- List itemWorkIndex (inventory slot) validated **0 .. 0x50 (80)** (`gcauction.c:351`).

**Listing fee flow.**
- Fee table = 81 (0x51) int32 entries, init to -1 (`ykWndAuction.cpp:2212`), indexed by **item category byte** (`YkAuctionFee::SetFee`, `ykWndAuction.cpp:46`).
- Client requests fee from server (`SendAuction_AskCommit`, action byte 4) and **blocks listing if player gil < fee** (`ykWndAuction.cpp:1681-1727`).
- Server relevance: server must compute/return the AH deposit in the AskCommit response and deduct on list (LotIn). Fee is per item-category, not per item id.

**Categories / sort (client-side search query, fed to `FFWCa*`).**
- Top-level category bytes: 0x21, 0x22, 0x23 (`ykWndAuction.cpp:2490,2507,2530`). Sub-category tables (`AucWepCateTbl`, `DAT_*`) are rodata, not in source — **INFERRED**.
- Sort codes observed: **2,3,4,5,6,7,8,9** (`ykWndAuction.cpp:2878-2966`); food/dish force code 9. Semantics in `FFWCa*` (out of scope).
- Server relevance: these are client display filters for the item-search service, not AH transaction packets; LSB does its own AH categorization.

**Packet format (0x4E AH request, 60-byte / 0x3c payload, wire id 0x4254).**
- `+4` action byte; `+5` box slot (0xff = all); `+8` int32 price/commission; `+0xc` u16 itemWorkIndex; `+0xe` u16 itemNo; `+0x10` int32 stacks.
- Action codes: AskCommit=4, Info=5, work=10, LotIn(list)=11, LotCancel=12, LotCheck=13, Bid(buy)=14, Get=15, Clear=16 (`gcauction.c:198..707`).
- Server relevance: maps to LSB's 0x04E "Auction House Action" packet; offsets and action codes are exact literals.

---

## 6. Party & alliance

- **Party = 6** (`ykWndParty.cpp:837` `iVar2 = 6`; cursor clamp `:372` `< 6`; wrap `:1061` `5 <`). Member stride 0x58 (88) bytes.
- **Alliance = 18 (0x12)** (`ykWndParty.cpp:835` default `iVar2 = 0x12`; index valid 0..0x11, `:909`; loops `< 0x12` `:991,1111`).
- **3 parties per alliance** (own + `Raid1` + `Raid2` widgets, `ykWndParty.cpp:103,129,148`); 18÷6.
- Group "Kind" byte: 0 = alliance, 1 = party, 2 = self/own-party (`gcgroup.c:140,382`). Group scratch table bound = 0x14 (20), stride 0x4c (76), per-kind region 0x5f4 (1524) — this 20 is a list cache, **not** the 18 member cap.
- Member name field = **16 bytes** (`gcgroup.c:41,45`; `RecvGroupList:1618`). Kick/change copies use 15 (`gcgroup.c:465,514,607`).
- Group opcodes (from `gcGroupInit` callbacks): recv 0xdc/0xdd/0xde/0xdf/0xe0/0xe1/0xe2/0xc8/0xcc; send solicit 0x6e, leave 0x6f, breakup 0x70, kick 0x71, change 0x73/0x77, solicit-res 0x74, list-req 0x76, checkID 0x78.
- Server relevance: PARTY_SIZE=6, ALLIANCE_MAX=18, 3 parties — match LSB. Opcodes/strides useful for aligning GP_SERV_GROUP_* handlers.

---

## 7. Search

- Filters present (each `FFWCaSetQuery*`): **Zone** (byte, `ykWndSearch.cpp:582`), **Area/region** (21 buckets via ZoneSortTbl, `:51`), **Job** (single byte, `:2229`), **Level min/max** (two bytes, `:2575`), **Race** (`:2287`), **Rank** (`:2200`), **Home nation** (byte, `:2258`), **Name** (text), **Comment / msg-type / LFP** (`:2351`), **Friend flag** (`:2151`), **Linkshell** (`:2160`).
- **Level clamp 1..99** (`ykWndSearch.cpp:2371,2382`) — this PS2 build supports level-99 era, not capped at 75. (The 25/50/75 in `ykWndParty.cpp:408-418` are HP/MP bar color thresholds, NOT level.)
- Job filter is a **single job byte** (not a bitmask) — one job per search.
- **Result status bitfield** (ushort at result `+0x26`, decoded in `EditLine`, `ykWndSearch.cpp:626`): `0x400` = recruiting / LFP (also sets group-mode `m_mode=1`, `:1665`); `0x10` = comment present; `0x8020` = away/anon; `(flags & 0xc0) >> 6` selects sub-icon; plus 0x4000/0x800/0x100/0x20/0x8000/0x2000/8 icon bits.
- **Search result buffer = 513 (0x201)** (`ykWndSearch.cpp:1184`); auction-history cap 14 (`:1200`); 15 lines shown (`:1394`); overflow → message 0xbd. The 513 is the client buffer ceiling, not a protocol guarantee — server may send fewer.
- Server relevance: maps to LSB search packets (0x0F4/0x0F5/0x0F6) — job, min/max level, zone, nation, rank, race, name, flags. The +0x26 ushort is the search flags word the server must populate (anon/away/LFP/comment).

---

## 8. Trade, bazaar, combine

**Trade.**
- Container = gil in slot 0 + **8 item slots** (10-entry backing array of 8-byte records; `TkTrade.cpp:343,472`; `gcitem.c:678` `< 10`). Per-slot: u32 ItemNum, u16 ItemNo, u8 ItemIndex.
- Gil = int32 (`TkTrade.cpp:467`), entered via CTkMoneyCtrl; no client clamp (server enforces gil/999,999,999 cap).
- Confirmation states `TKTRADESTATUS` (CTkTrade.h: m_LocalStatus +0xc0, m_RemoteStatus +0xc4): **0=idle, 3=trading/list-active, 4=accepted**. Both at 4 ⇒ exchange. Any edit resets both to 3.
- Opcodes: request 0x32, action 0x33 (sub-codes 0=start, 1=cancel, 2=accept/commit, 3=un-accept; `gcitem.c:551-647`), my-list item 0x34. Duplicate-item guard `gcitem.c:679`.
- Trade range check = **36.0 = 6.0² yalms** (`TkTrade.cpp:1835,1843`).

**Bazaar.**
- Price = **uint32 per unit** (`gcbazaar.c:251,264`); item index 1 byte. No client max. List entry stride 44 bytes; region 0xdec (3564). Per FFXI, bazaar cap is **7 items** (settable count is server-enforced; client streams one item at a time via `gcBazaarItemSet`, no client count check).
- Regional tax: `gpBazaarGetTax(basePrice, num, taxRate)` — tax rate **u16** at zone+0x2511e (`gcbazaar.c:358`); total = `basePrice*num + tax` (`gpbazaar.c:105`).
- Opcodes: list req 0x105, buy req 0x106 (BuyNum u32), item set 0x10A, close 0x10B.

**Combine / synthesis.**
- **1 crystal (separate header field) + up to 8 ingredients** (`ykWndCombine.cpp:155` `if (7 < count)`; crystal `:149`; recv loops `< 8`, `gccombine.c:161`). UI selection arrays hold 10, but wire caps materials at 8.
- ASK packet (opcode 0x96, len 0x22=34): `+4` checksum byte `((item0+7)*(...)*(matCount+5)) % 0x7f`, `+6` crystal u16, `+9` matCount, `+0xa` 8×u16 item nos, `+0x1a` 8×u8 inv indices. Item array descending-sorted before send (`EncodeCombine`, `gccombine.c:28`).
- Client cooldown: 10 s normal / 3 s GM (`gccombine.c:242`), 30 s request timeout.
- Server relevance: synthesis = crystal + 8 ingredients (matches LSB). Servers should not rely on slot order (client sorts), and the +4 checksum is advisory.

---

## 9. Chat, linkshell, tell

- **Max message body = 150 bytes (0x96)** for ALL channels (say/shout/tell/party/linkshell) — single clamp in `gcChatStdSend` (`gcchat.c:112`) and `gcChatNameSend` (`:182`); receive clamps to 150 into a 160-byte (0xa0) buffer (`:218,228`). No per-channel length, no client send rate-limit.
- **Channel Kind byte** (`gcchat.c` switch `:243-396`): 0/1 = Say, 3 = Tell, 4 = Party (requires party ≥ 2, `:105`), 5 = Linkshell (requires active LS, `:96`), 6/7 = Shout/Yell, 8 = Emote, 0xc = GM tell, 0x0d-0x17 = system messages.
- Send opcodes: standard chat **0xB5** (header 8B), named/tell **0xB6** (name at +5, body at +0x14), recv callback **0x17**.
- **Linkshell name UI input = 15 chars (0xf)** (`ykWndComlink.cpp:1367`). (Item-name display buffer is 20 bytes / 0x14 — a separate field; note the discrepancy.)
- **Linkshell slots = 3 menus, 1 active for chat** (`ykWndComlink.cpp:316-361`); `gcGroupComlinkActiveGet` returns the one active LS. The "3 kinds" likely encode LS color/pearl variants — **INFERRED** semantics; treat as ≥ standard LS1-active model.
- **No macro buffers/limits** in these modules (macros live elsewhere — CERTAIN absence in scope).
- Server relevance: 150-byte body matches canonical FFXI; LSB inbound handlers for 0xB5 (standard) and 0xB6 (named/tell) switch on the Kind byte at +4. No client flood protection — server must implement it.

---

## 10. Friend list & blacklist

- **In-game blacklist = 100 entries** (`gcblklist.c:122,138,192,245`), 24-byte stride (4-byte ID + 15-byte name). Load opcode 0x41, edit 0x42, query 0x3c, edit-send 0x3d. **60-second refresh throttle** (`gcblklist.c:345` etc.). Matches FFXI's 100-entry blacklist.
- **POL friend list = 300 = 200 normal + 100 "black"** (`gcfriend.c:1558-1562`; checks `< 200` / `< 100` at `:473,487`). This is the **PlayOnline-account** layer (`POL_FRIEND_*`), distinct from in-game features — LSB does not implement it; relevant only for POL friend sync. Friend update batch = 20/cycle (`gcfriend.c:270`).
- **Character name = 15 chars + NUL (16-byte field)** (`gcchat.c:177` send guard `< 0x10`; recv `strncpy(...,0xf)` `:220,233`; blacklist 15-byte entries). Matches LSB `chars.charname varchar(15)`.

---

## Certain vs inferred — quick recap

- **CERTAIN (code literals / DWARF):** inventory 81/0x51, 3 containers, 16 equip slots, RARE=0x8000 / EX=0x4000, item ID u16, qty u32, party 6, alliance 18, auction 8-slot/7-sell, price 1..999,999,999, bid stacks 0..99, fee table 81×int32, trade 8 item slots + gil, combine 1 crystal + 8 ingredients, chat body 150, recv buf 160, blacklist 100, blacklist throttle 60 s, POL friends 200+100, name 16-byte, LS name input 15, level clamp 1..99, search buffer 513, 8 visual model slots; all cited packet opcodes and field offsets.
- **INFERRED:** specific 16-slot name ordering (count certain, name strings in undecompiled rodata `equippostbl`); auction sub-category table contents (`AucWepCateTbl`/`DAT_*`); property bits beyond RARE/EX; bazaar 7-item cap (server-enforced, no client literal); LS "3 kinds" semantics; auction sort-code meanings (in `FFWCa*`).
