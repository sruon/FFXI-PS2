# 07 — Lobby/Auth Handshake & Bug-Report Telemetry

Supplementary review of the lobby (account/character) login protocol and the
client's built-in bug-reporting/telemetry service. These sit *outside* the
zone/game protocol but are exactly what an emulator's **login/lobby server**
must speak.

Source files:
- `src/main/net/lobby_clt/nttcpmakepacket.cpp` — lobby packet build/parse
- `src/main/net/lobby_clt/ntlogin.cpp` — lobby DLL request API
- `src/main/net/game_cli/gclogin.cpp` — client-side login state machine
- `src/main/net/bug_db/FDtFFXi/fdtFFXiService.cpp` — bug-report uploader

---

## A. Lobby packet framing — `lpkt_header` + chained MD5 integrity

`LobbyPktCreate` / `LobbyPktIdentiferCheck` / `LobbyPktCommandGet`
(`nttcpmakepacket.cpp:16-308`).

The lobby wire format is a fixed little header followed by command-specific
payload. Reconstructed from the build/parse code:

| Offset | Field | Notes |
|---|---|---|
| `+0x00` | `uint size` | total packet length in bytes (e.g. `0x44`, `0x60`, `0x88`, `0x90`) |
| `+0x04` | `uint magic` | always `0x46465849` = ASCII **"FFXI"** (`nttcpmakepacket.cpp:120`) |
| `+0x08` | `uint command` | sign bit (`0x80000000`) is a flag; low bits are the command id, masked `& 0x7fffffff` (`:27`, `:400-401`) |
| `+0x0c` | `byte[16] md5` | MD5 digest of the whole packet, computed with this field zeroed |
| `+0x1c…` | payload | per-command |

**Integrity check** (`LobbyPktIdentiferCheck`, `:43-68`): only validates when
`0x1b < size < 0xfc1` (i.e. 28..4031 bytes — a hard packet-size window the
server must respect). It saves the 16-byte digest, zeroes that field,
`MD5Update(header, size)` over the whole packet, and `memcmp`s. Same pattern is
applied on the *outbound* side after each `LobbyPktCreate` branch
(`:139-145` etc.).

**Server relevance:** an emulator's lobby server must (1) emit the literal
`"FFXI"` magic at +4, (2) embed an MD5 of the zeroed-digest packet at +0x0c,
and (3) keep every lobby packet within 28–4031 bytes or the client silently
drops it (`LobbyPktCommandGet` returns 0 → "no command"). This is a concrete,
testable framing contract.

## B. Rolling MD5 "next key" auth chain

`rapMakeMD5` / `rapGetMD5NextKey` (`nttcpmakepacket.cpp:321-366`) and the parse
side in `ntLoginAnalyzePacket` (`:379-480`).

`lpkt_work+0x138` holds a running **key counter**. Behaviour is gated by a flag
at `lpkt_work+0x134 & 1`:
- **Flag clear (0):** key is just `prev + 1` (`:329`, `:406`, `:439`), and
  `rapMakeMD5` simply `memcpy`s the 16-byte password block verbatim
  (`:356-357`) — i.e. *no* hashing. Looks like the debug/plaintext path.
- **Flag set (1):** key advances via `GetMD5NextKey(x)` and the password is run
  through `makeMD5(key, passwd, digest)` / `makeMD5len(..,0x10)`
  (`:359-364`). This is the live salted-auth path.

Every outbound command re-derives and stores the next key
(`*(pwork+0x138) = rapGetMD5NextKey(...)`), so **request N's MD5 depends on the
key state left by request N-1** — a sequence-locked handshake.

**Server relevance:** the lobby auth is a stateful MD5 key-ratchet, not a
stateless per-packet hash. An emulator must mirror the `+0x138` counter
advancement in lock-step or every post-login command fails its digest. The
`0x134 & 1` flag distinguishes a plaintext-password mode from the hashed mode —
worth checking which retail/private setups used.

## C. Lobby command id map (outbound, from `LobbyPktCreate`)

Each `command ==` branch builds a distinct request. Ids and fixed sizes:

| cmd | size | payload (offsets into outbuf) | inferred purpose |
|---|---|---|---|
| `0x07` | `0x44` | accountId@+0x1c, worldId@+0x20(u16), name[15]@+0x24, pw-md5@+0x34 | character op (select/commit?) |
| `0x14` | `0x34` | accountId, worldId(u16), pw-md5@+0x24 | character op |
| `0x1f` | `0x2c` | pw-md5@+0x1c | simple authed request |
| `0x21` | `0x90` | accountId@+0x1c, pw-md5@+0x20, 0x60-byte blob@+0x30 (char appearance/create data) | **create / commit character** |
| `0x22` | `0x60` | accountId, pw-md5, name[15], blob[15]@+0x40, blob[15]@+0x50 | rename / GM op |
| `0x24` | `0x2c` | pw-md5@+0x1c | simple authed request |
| `0x26` | `0x88` | name[15]@+0x1c, 8-byte@+0x2c, 0x40-byte@+0x34, 15-byte@+0x74, u32@+0x84 | **initial login** (account id + key material) |
| `0x28` | `0x44` | accountId, worldId(u16), name[15], pw-md5 | character op |
| `0x2b` | `0x44` | accountId, u32@+0x20, name[15], pw-md5 | character op |

(Field offsets above are byte offsets; the code indexes `outbuf` as `uint*` so
`outbuf+7` = byte +0x1c, `outbuf+9` = +0x24, etc.)

**Inbound** (`ntLoginAnalyzePacket`, command in `inbuf+8 & 0x7fffffff`):
- `0x23` — **character list**: count@`+0x1c`, then `count` × 0x14-byte entries
  (`u32 id` + `16-byte name/digest`) copied to `pwork+0xa64…` (`:404-420`).
- `0x20` — **world/character data**: count@`+0x1c`, then `count` × **0x8c-byte**
  records copied to `pwork+0x148` (`:448-464`). 0x8c (140) bytes is the
  per-character lobby record size.
- `0x0b` — session params: four u32 at `+0x38…+0x44` → `pwork+0x124…0x130`
  (`:433-437`).
- `0x05` — **key sync**: server sends a key at `+0x1c` that overwrites the local
  `+0x138` counter, plus a value at `+0x20`→`pwork+0x1a20` (`:465-478`).
- `0x03` — ack/no-op (just advances the key).

**Server relevance:** this is a near-complete spec of the lobby protocol — id
list, the 0x8c-byte character record, and the key-sync packet `0x05` that
*resets* the client's MD5 ratchet. Emulator lobby servers can be validated
field-for-field against this.

## D. Client login state machine (`gclogin.cpp`)

`gcLoginProc` and friends enumerate the full client flow:
`Cliinit → Dataload → ChrList → ChrSelect/Create/Delete/Rename → Zone →
FFXIout/Ragout`. Notable handlers:
- `gcLoginMoveGMChrProc` (`:1391`) and `ntTcpDLLRequestMoveGMChr`
  (`ntlogin.cpp:711`) — a **GM character-move** request exists in the retail
  client lobby API.
- `gcLoginRenameChrProc` + `gcLoginGetRenameFlag` (`:1280`, `:2295`) — rename is
  a server-driven flag; the client only offers rename when the server sets it.
- `gcLoginZonechangeProc*` (`:1977-2058`) and `gcLoginFFXIoutProc*` /
  `gcLoginRagoutProc` (`:2088-2224`) — zone-in vs full logout ("ragout") vs
  "FFXIout" are distinct lobby transitions.

`ntlogin.cpp` exposes the DLL surface: `RequestConnection`, `RequestLobbyLogin`,
`RequestGetChr`, `RequestSelectChr`, `RequestCreateChr(Pre)`, `RequestDeleteChr`,
`RequestRenameChr`, `RequestQueryWorldList`, `RequestKeyIncrement`,
`SetAthCode`/`SetClientCode`/`SetExCodeClient`/`GetExCodeServer`/`SetPasswd`
(`ntlogin.cpp:122-1456`). The `AthCode`/`ClientCode`/`ExCode` split is the
authentication-token plumbing between PlayOnline and the lobby.

**Server relevance:** confirms the lobby supports GM moves and server-gated
renames, and names every request the client can issue — a checklist for lobby
emulation. `RequestQueryWorldList` implies multi-world support in the protocol.

## E. Built-in bug-report / telemetry uploader (`bug_db`)

The retail client ships an entire embedded database/reporting library
(`LibFDt`, `LibGN`, `FDtFFXi`). `fdtFFXiService.cpp` builds and uploads
structured reports:
- `FDtSendReport(repType)` dispatches type 1/2/3 → `FDtSendReportX`
  (`:895-916`).
- `BuildReportPathBlock` / `BuildReportAttrLine` (`:435-505`) serialize a
  hierarchical "path + attribute lines" record; `BuildReportAttrLine` caps each
  line at `0xf4` (244) bytes (`:807`).

**Server relevance:** mostly historical/forensic, but worth flagging: a live
retail client had a channel that uploads structured client-side reports
(likely GM bug reports / `/bug`). Not needed for emulation, but explains the
`LibFDt`/`LibGN` bulk in `net/` and is a potential source of additional
opcodes if any reporting packets share the game socket.

---

### Cross-checks worth doing against LSB
1. Confirm LSB's `login` server emits the `"FFXI"` magic + MD5 framing and the
   0x8c-byte character record (cmd `0x20`).
2. Verify the key-ratchet: does LSB increment a per-session counter the way
   `+0x138` does, or does it run the plaintext (`0x134 & 1 == 0`) path?
3. Check whether LSB implements the GM-move (`0x...`) and server-gated rename
   flags, since the client has UI paths for both.
