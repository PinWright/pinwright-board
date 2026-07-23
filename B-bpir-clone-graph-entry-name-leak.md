---
id: B-bpir-clone-graph-entry-name-leak
title: "BPIR decompiler leaks clone graph name (EdGraph_N) into entry signatures for composite-bearing graphs"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, decompiler, asset-dump, entry-signature, composite]
encounters: 1
lastSeen: 2026-07-23T00:00:00Z
---
# BPIR decompiler leaks clone graph name (EdGraph_N) into entry signatures for composite-bearing graphs

`FBpirDecompiler::DecompileGraphInternal` (`Source\PinWright\Private\Decompiler\BpirDecompiler.cpp:560`)
clones composite-bearing graphs before decompiling — `CloneGraph` outered to
the source `UBlueprint` (`:593-613`). The clone gets an auto-suffixed name
(`EdGraph_7`, `EdGraph_9`, ...). `FBpirTextEmitter::EmitEntrySignature` then
derives the entry name from `EntryNode->GetGraph()->GetName()`
(`BpirTextEmitter.cpp:1342-1344` function branch, `:1456-1457` macro branch),
so the clone's transient name is emitted instead of the source graph's name.

Symptoms:

- **Asset-dump mirror noise** — 6 `/App` selector widgets flap in every dump
  commit (`entry function EdGraph_7` ↔ `entry function EdGraph_9`, e.g.
  `W_MeteoSelector`, `W_DronSelectionArrows`), confirmed 2026-07-23. The
  suffix depends on transient object-name allocation, so every sweep produces
  a spurious diff.
- **Silent correctness bugs for cloned graphs** — override detection
  (`:1385-1403`) keys off the clone name, so `entry override` is never
  emitted; the `UserConstructionScript` check (`:1347`) misclassifies; macro
  entry names come out suffixed. The caller trusts an entry signature that is
  a lie.

Severity rubric: silent wrong data, but reach is limited to composite-bearing
graphs and the practical impact is dump-mirror noise plus a missing `override`
keyword — not a broad correctness break — Medium.

**Workaround:** none for the emitted text; for dump-mirror churn, ignore
`entry function EdGraph_N` diffs on composite-bearing widgets.
**Fix:** (in flight, same-day) thread the source graph name from
`DecompileGraphInternal` into `EmitEntrySignature` as a parameter; bump
`bpir.txt` aspect version 4→5 so cached dumps regenerate; regression tests in
`Tests\Bpir\TestBpirCompositeEntryName.cpp`.

## History
- `#1-initial-repro` `OPEN` reporter — Clone graph's auto-suffixed name (`EdGraph_N`) leaks into BPIR entry signatures for composite-bearing graphs: dump-mirror flapping on 6 /App selector widgets (`EdGraph_7` ↔ `EdGraph_9`), override detection keyed off clone name so `entry override` never emitted, UserConstructionScript check misclassifies, macro entries suffixed. Root cause `DecompileGraphInternal` clone (BpirDecompiler.cpp:560,593-613) + name derivation from `EntryNode->GetGraph()->GetName()` (BpirTextEmitter.cpp:1342-1344, 1456-1457).
- `#2-source-name-threaded-aspect-5` `IN-REVIEW` developer — Fixed in plugin commit `d3a942d0`: source graph name threaded from `DecompileGraphInternal` into `EmitEntrySignature` so cloned composite-bearing graphs emit the source graph's name (function and macro branches); `bpir.txt` aspect version bumped 4→5 so cached dumps regenerate. Full automation suite clean on UE 5.8: 4 new regression tests in `Tests\Bpir\TestBpirCompositeEntryName.cpp` pass (function-entry name, determinism under name-counter perturbation, macro entry name, override keyword survival); BpirAspectVersion pin test updated 4→5, re-verified green.
