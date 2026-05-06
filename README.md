# FFXI-PS2

Decompiled source of the Final Fantasy XI PS2 client (`SCUS_972.66`,
MIPS R5900, MetroWerks CodeWarrior, 2003).

Read-only artifact. Not buildable.

## Layout

```
src/                  function bodies, organized as in the original source tree
  main/
    actor/            xiactor, xisklactor, xiatelnet, commandcalc, xievent, ...
    net/
      bug_db/, engine_net/, game_cli/, game_prot/, lobby_clt/, sys_ps2/
    miyagawa/         ymmem, ymsound, ymschdecript, ...
      effect/
    fshira/           FsConfigMenu, XiFileImpl, fstextinput, ...
    yk/               PS2 UI (auctions, party, friends, ...)
    tk/               UI primitives
    akaza/, dancer/, boot/, faq/, km/, zone/
  ohno/               PS2 graphics (ko_vu1, ko_vum, ko_*)
  kazumi/             animation / transforms (KzMod, KzMotionQue, ...)
  common/             shared
  vendor/             Metrowerks runtime, SCE EE crt0
  _unmapped.cpp       3,104 functions without DWARF compile-unit info

include/              one header per recovered class (declarations only)
_globals.h            1,763 typed globals
_types.h              127 enums
index.md              navigation
```

The directory hierarchy under `src/` is the original SE source tree,
recovered from the Windows paths (`C:\home\miyagawa\src\FFXi\FFXi_PS2\...`)
in the binary's debug symbols.

## How it was made

1. **Disassembly**: Ghidra 12.0.4 with [ghidra-emotionengine-reloaded](https://github.com/chaoticgd/ghidra-emotionengine-reloaded)
   for PS2 EE-MIPS (R5900 + MMI + VU0/VU1).
2. **Type recovery**: the binary's `.debug` section is 14.8 MB of MetroWerks
   DWARF 1.1 with original field names, parameter names, types, source
   paths. Parsed with [`encounter/decomp-toolkit`](https://github.com/encounter/decomp-toolkit)
   (`dtk dwarf dump`).
3. **Round-trip**: DWARF struct layouts and `this` types applied back into
   Ghidra, then re-decompile. Bodies emit `this->fieldname` instead of
   `*(undefined4 *)(this + 0x20)`.
4. **Protocol semantics**: 60 server packet handlers cross-referenced
   against [atom0s/XiPackets](https://github.com/atom0s/XiPackets).
5. **LLM fallback rename**: parallel agents fill name gaps for classes /
   offsets / functions DWARF doesn't cover. Tagged `[llm]` or `[synth]`
   so it's clear which names are inferred vs authoritative.

## Naming provenance

Each field in `include/<Class>.h` shows its source:

| Source | Confidence | Tag |
|---|---|---|
| `mdebug` | from DWARF debug symbols | (no tag) |
| `llm` | LLM inferred from access patterns | `[llm]` |
| `synth` | offset known, name is `field_0xNN` placeholder | `[synth]` |

Function parameters with real names came from DWARF; remaining `param_N`
slots are uncovered.

## Coverage

- 11,042 decompiled functions
- 469 source files (compile units), 689 class headers
- 9,529 / 11,261 class fields with authoritative DWARF names (~85%)
- 27,146 `this->fieldname` accesses vs 819 remaining `*(undefined4 *)(this + 0x..)` casts

## Identity

`SCUS_972.66`, sha256 `d4622c1cc4a3012c…`, image_base `0x00100000`.

## Not a leak

Generated from publicly available retail binaries with standard RE
tools. No SE source was used.
