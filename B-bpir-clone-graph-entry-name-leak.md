---
id: B-bpir-clone-graph-entry-name-leak
title: "BPIR decompiler leaks clone graph name (EdGraph_N) into entry signatures for composite-bearing graphs"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, decompiler, asset-dump, entry-signature, composite]
encounters: 2
lastSeen: 2026-09-02T00:00:00Z
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
- `#2-source-name-threaded-aspect-5` `IN-REVIEW` developer — Fixed in plugin commit `678fb57c`: source graph name threaded from `DecompileGraphInternal` into `EmitEntrySignature` so cloned composite-bearing graphs emit the source graph's name (function and macro branches); `bpir.txt` aspect version bumped 4→5 so cached dumps regenerate. Full automation suite clean on UE 5.8: 4 new regression tests in `Tests\Bpir\TestBpirCompositeEntryName.cpp` pass (function-entry name, determinism under name-counter perturbation, macro entry name, override keyword survival); BpirAspectVersion pin test updated 4→5, re-verified green.
- `#3-orphan-warning-source-name` `IN-REVIEW` developer — A repeated 8,267-asset `/App` dump found one residual instance of the same bug: `BP_RobotHend` orphan warnings alternated between `EdGraph_14` and `EdGraph_15` because `DecompileGraphInternal` still formatted that diagnostic with `WorkingGraph->GetName()`. Switched the warning to the already-captured `CurrentSourceGraphName` and bumped `bpir.txt` aspect 5→6. Live Coding compiled successfully; two targeted dumps emitted `EventGraph` and the same SHA-256 (`DE440DDB9C378EE5A4D17E15D27CD7384011893CEF8123919D44D8BF041AA364`), then a forced `/App` pass was byte-identical to its 29,619-file pre-pass snapshot. No automation suite was run.
- `#4-mirror-wide-evidence-no-leak-remains` `IN-REVIEW` reporter — Mirror-wide field evidence, **not a verification**; status and `encounters` deliberately unchanged (corroboration, not a re-hit). After a forced full `/Game` + `/App` sweep of the PDS project (2026-09-02, plugin 0.7.0, UE 5.8.1, 30,807 dump dirs), **0 of 1,701 committed `bpir.txt` files contain the string `EdGraph_`** — including the `BP_RobotHend` orphan-warning case `#3` fixed. The churn this ticket was filed for is likewise no longer dominant in the mirror: `bpir.txt` accounted for 47 of the 40,597 files the refresh modified, and the sampled diffs carried no `EdGraph_` lines. That is not the flap test (a flap needs two sweeps compared; this is one forced sweep against the committed baseline), so it does not close the ticket — it does establish that no residual leak survives anywhere in a 1,701-file corpus that previously flapped on 6 `/App` selector widgets.
- `#5-fix-survived-upstream-pull` `IN-REVIEW` reporter — Bookkeeping only: the host plugin clone was pulled 398 commits forward (`b16f0f2b` -> `347826a6`) after `#4` was written, so this confirms the pull is not a confounder for the pending flap test. Status, severity and `encounters` unchanged. Both fix commits are ancestors of `347826a6` and both halves are present at HEAD: `FBpirDecompiler::DecompileGraphInternal` captures `CurrentSourceGraphName = Graph->GetName()` before cloning (`BpirDecompiler.cpp:574`, with the rationale comment at `:583`) and threads it into `TextEmitter->EmitEntrySignature(EntryNode, TargetBlueprint, CurrentSourceGraphName)` (`:686`); the emitter takes it as a parameter (`BpirTextEmitter.cpp:1146`) and uses it in both branches — `FString GraphName = SourceGraphName;` for functions (`:1344`) and `FIrTextUtils::FormatNameToken(SourceGraphName)` for macros (`:1457`) — so no branch reads `EntryNode->GetGraph()->GetName()` any more. The `#3` orphan-warning half is also intact: the diagnostics at `BpirDecompiler.cpp:982` and `:1476` format with `*CurrentSourceGraphName`, not `WorkingGraph->GetName()`. The aspect pin carries the `#3` bump: `{ TEXT("bpir.txt"), 6 }` in the `GetAspectVersion` table (`AssetDumpCache.cpp:750`). No commit in the pulled range reverts any of it.
