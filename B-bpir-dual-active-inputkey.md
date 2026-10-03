---
id: B-bpir-dual-active-inputkey
title: "BPIR decompilation silently drops one execution branch from InputKey nodes with both Pressed and Released connected"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, input-key, pressed, released, round-trip]
encounters: 2
lastSeen: 2026-09-29T13:00:00Z
---

# BPIR drops one branch from a dual-active InputKey node

## What happens

The decompiler chooses one initial exec pin for each entry node. For an
`UK2Node_InputKey`, an active Released pin replaces that choice, and only the
resulting chain is walked
(`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp:847-893`). The text
emitter likewise returns one signature per node and chooses `key_released` whenever
Released is connected (`BpirTextEmitter.cpp:1805-1824`). Entry collection records
the physical InputKey node once, not once per active exec pin
(`Handlers/Blueprint/BlueprintHandlerUtils.cpp:2391-2395,2564-2569`).

When both Pressed and Released are connected on the same node, the Released body is
selected and the Pressed body has no emitted entry. The BPIR output succeeds but is
not a complete representation of the graph.

## Why it matters

Round-tripping that output can lose input behavior silently. Severity is Medium:
silent omission is serious, but the dual-active legacy InputKey shape is narrow and
has a practical authoring workaround.

## What should happen

Model InputKey entries by node plus active exec pin. Emit independent
`key_pressed` and `key_released` entries and walk each connected body exactly once,
with regression coverage for one node carrying both branches.

## Workaround

Use separate InputKey nodes for Pressed and Released, or inspect and repair the
decompiled BPIR manually before compiling it elsewhere.

## Related

- `B-bpir-key-released-entry-silently-dropped` — wave-6 ticket fixed the
  released-only traversal and explicitly left simultaneous Pressed/Released support
  outside its scope.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed one-node entry collection, Released-over-Pressed exec selection, and one emitted signature at `BpirDecompiler.cpp:847-893`, `BpirTextEmitter.cpp:1805-1824`, and `BlueprintHandlerUtils.cpp:2391-2395,2564-2569`. No live Blueprint reproduction, build, test, editor, or MCP call was run. Severity Medium because the silent round-trip omission is limited to a dual-active legacy InputKey node and separate nodes are a workaround.
- `#2-live-repro-level-editor-lmb` `OPEN` reporter - Live evidence, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv` (Linux), plugin `8fcc0b2a`. In `/App/App/LevelBlueprints/B_LevelEditorCharacter`, the `Left Mouse Button` and `Shift Left Mouse Button` `K2Node_InputKey` nodes each wire Pressed -> Branch(IsDraggingLocalActor) -> `PlacePrePlacedActorFromClick` / `FindAndGrab`, and Released -> `Release`. The checked-in dump (`asset-dumps/App/App/LevelBlueprints/B_LevelEditorCharacter/bpir.txt`, produced by the `asset.dump` sweep) renders both nodes as `entry key_released LeftMouseButton()` containing only `call Release(...)`. The Pressed chain, including the only `FindAndGrab` call in the project, is absent, and nothing marks the omission. A dump from July (commit ad62013d7b) showed the opposite half: the Pressed body under a `key_released` label. The live `blueprint.graph.get_execution_flow {startNodeId}` showed both pins correctly. This matters in practice. A map-editor selection bug came down to `FindAndGrab` running on press, and the C++ gesture code had been written on the assumption that it ran on release. Moderate: one extra live-graph pass to settle which pin calls what.
- `#3-entry-per-active-pin` `IN-REVIEW` developer - Still reproducible in current source before the fix (`DecompileGraphInternal` forced the Released pin and `EmitEntrySignature` picked `key_released` whenever Released was wired). `DecompileGraphInternal` (`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`) now expands entry points into (node, exec pin) pairs: an InputKey node yields a `Pressed` entry when Pressed is wired (or when neither is), plus a `Released` entry when Released is wired, and each walks from its own pin. `FBpirTextEmitter::EmitEntrySignature` takes an optional `EntryExecPin` and names `key_pressed` / `key_released` from it (`BpirTextEmitter.h/.cpp`). Entry collection in `BlueprintHandlerUtils` is unchanged (the decompiler is the only consumer that needed the pin). `bpir.txt` aspect 11 -> 12. Wiki: `docs/wiki-src/bpir.examples.key-input.md`. Test `PinWright.bpir.input_key_dual_active.EmitsBothEntriesAndRoundTrips` (`Tests/Bpir/TestBpirInputKeyDualActive.cpp`): folds a compiled released body onto the pressed node so one node has both pins wired, asserts separate `key_pressed SpaceBar()` / `key_released SpaceBar()` blocks each with only its own body (fails unfixed), then compiles the output into a fresh Blueprint (two nodes) and asserts a byte-identical decompile. Compiler-side gap found while here and filed: `B-bpir-replace-dual-active-inputkey-leaves-bare-node` (a `Replace` compile of both entries over the original dual node leaves the node unwired).
- `#4-run1-entry-tie-order` `IN-REVIEW` developer - run1 failure of `PinWright.bpir.input_key_dual_active.EmitsBothEntriesAndRoundTrips`: the round-trip decompile emitted `key_released` before `key_pressed`. Root cause was not the emitter (one dual node always expands Pressed then Released) but `FGraphWalker::FindEntryPoints` (`Source/PinWright/Private/Decompiler/GraphWalker.cpp`): it sorted entries with the unstable `TArray::Sort` on (graph path, NodePosY). The round-trip Blueprint holds two InputKey nodes at the same `@(0, 864)`, and UE's IntroSort uses a selection sort for ranges of 8 or fewer that swaps the first maximum to the back, so two equal entries came out reversed. The result was deterministic but undefined, and it reversed on every round trip. Now `StableSort`: tied entries keep the graph's node order, and compile creates entry nodes in text order. That ordering is documented in `docs/wiki-src/bpir.entry-points.md`, and the key-input example says pressed comes first. The `bpir.txt` 12 comment in `AssetDumpCache.cpp` covers the byte change, and CHANGELOG has a Fixed line. The existing round-trip assertion pins it; it fails with the fix reverted, as run1 showed. fastcheck OK on GraphWalker.cpp, AssetDumpCache.cpp and TestBpirInputKeyDualActive.cpp; check_test_ids and check_test_skips are CLEAN.
- `#5-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux. `PinWright.bpir.input_key_dual_active.EmitsBothEntriesAndRoundTrips` passed non-skipped in run3/full. That is the run1 failure, fixed by the GraphWalker StableSort in #4. The test builds one InputKey node with both Pressed and Released wired, asserts separate `key_pressed SpaceBar()` / `key_released SpaceBar()` blocks each holding only its own body, then compiles into a fresh Blueprint and asserts a byte-identical decompile. This meets the ask: an entry per active exec pin, each body walked once, with regression coverage. Aspect bump: `PinWright.AssetDumpCache.BpirAspectVersion` passed, so stale `bpir.txt` caches such as B_LevelEditorCharacter refresh. Limits: the PDS B_LevelEditorCharacter dump was not regenerated in this run. The compiler-side Replace gap is tracked separately as `B-bpir-replace-dual-active-inputkey-leaves-bare-node`.
