---
id: B-bpir-decompiler-emitter-coverage-gap
title: "BPIR decompiler: emit node_props { ... } for non-default UPROPERTY values on generic K2Nodes (residual after sub-tickets)"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, emitter-coverage, generic-fallback]
---

# BPIR decompiler emitter coverage gap — residual node_props emit

This umbrella was split into implementation tickets in history
`#3-implementation-subtickets-filed`. The pin-walked args/exec emit
already lives in `Decompiler/BpirTextEmitter.cpp::EmitGenericNode`
(via `FormatArgs` + `FormatExecTargets`). The matching compile-side
surface for `node_props { ... }` parsing landed via
`F-bpir-add-generic-node-statement-form`, with the property-vs-
AllocateDefaultPins ordering rule provided by
`B-bpir-compile-property-vs-allocate-pins-ordering`.

The residual scope owned by this umbrella is the decompile-side
`node_props { ... }` emit: walk the K2Node's UPROPERTY values against
its CDO and append the diff to the existing `call K2Node_<TypeName>(args)`
output so non-pin state round-trips through the same generic path.

## Residual scope (this ticket)

`Decompiler/BpirTextEmitter.cpp::EmitGenericNode` ends with
`call K2Node_<TypeName>(args)<execTargets>`. Insert an optional
`node_props { Key: Value, Key2: Value2 }` block between `(args)` and
the exec clause, built from the existing CDO-diff helper at
`Utils/PropertyUtils.cpp::BuildSparsePropertyDiffJson` (filters
`CPF_Edit | CPF_BlueprintVisible`, skips `CPF_Transient | CPF_Deprecated`,
deep-compares against the CDO, sorts results by name).

The keyword stays `call` — no `generic` opcode, no parser dispatch
swap, no tokenizer entry. The existing `K2Node_<TypeName>` heuristic
in `BpirCompiler.cpp` already routes back through the same fallback.

The JSON-to-BPIR value formatter lives inside the
`BpirStructLiteralUtils` namespace (consolidated by
`E-bpir-struct-literal-formatter-duplicated`) so struct values reuse
the existing `ExportText`-based serialisation rather than growing a
parallel formatter on the emitter side.

## Where typed emitters still earn their keep

Unchanged from the original analysis — typed emitters remain
justified for nodes with non-pin runtime state the property walk
misses (AsyncAction proxy class), nodes with dynamic pin sets that
`AllocateDefaultPins()` + `ReconstructNode()` can't replay, and
sugar opcodes that are dramatically more readable than the generic
form (`if`, `while`, etc.).

## Audit data (informative, not a fix scope)

These were the K2Node types in the cache when filed. They are
**examples**, not the spec — the fix must cover unknown types too:

| Node type | Files | Notes |
|-----------|-------|-------|
| `K2Node_CreateWidget` | 49 | class + `OwningPlayer` |
| `K2Node_SpawnActorFromClass` | 29 | class, transform, exposed-on-spawn |
| `K2Node_GenericCreateObject` | 18 | outer + class |
| `K2Node_SetFieldsInStruct` | 16 | struct + field assignments |
| `K2Node_VariableSetRef` | 15 | ref target |
| `K2Node_PlayAnimationTimeRange` | 13 | montage + range |
| `K2Node_GetDataTableRow` | 12 | table + row name |
| `K2Node_LatentAbilityCall` | 6 | ability + payload |
| `K2Node_AssignmentStatement` | 5 | local-ref target |

Validated Get (`K2Node_VariableGet` with exec pin) is filed separately
in `B-bpir-validated-get-loses-identity` because it has a one-line
fix in the existing variable-get emitter, not the generic path.

**Repro:** any BP using one of the above; output is
`call K2Node_<Type>()` with no args/edges.

**Workaround:** Live `blueprint.inspect` reveals the real pins; BPIR
text is the only degraded surface.

## History
- `#1-initial-audit` `OPEN` reporter — Cross-cache scan of 2,210 `bpir.txt` files found 9 K2Node types in the generic fallback. Same root cause as tracked AsyncAction issues; filed as one umbrella because the fix shape was assumed identical to per-type emitters.
- `#2-rescoped-to-general` `OPEN` reporter — Rescoped: per-type typed emitters are the wrong fix because the long tail of K2Node subclasses (UE 5.x has ~80; plugins add dozens more) will keep producing new instances of this bug. Fix is a pin-driven reflective generic emitter on the decompile side plus a matching `generic K2Node_<Type>(...)` instantiate-and-replay path on the compile side, so any K2Node round-trips without per-type code. Acceptance: every K2Node in engine + loaded plugins round-trips through generic or an explicitly-justified typed emitter; nothing falls through to a warning-only stub.
- `#3-implementation-subtickets-filed` `OPEN` reporter — 5-agent codebase audit of BpirDecompiler / BpirTextEmitter / GraphWalker / BpirCompiler / BpirParser / BpirTokenizer / BpirTypeGrammar / CodeNodeEmitter / CodePinResolver / CodeFunctionResolver / BpirValueResolver / BpirSubgraphCompiler / NodeLayoutEngine confirmed the gap and split it into implementation tickets: `F-bpir-add-generic-node-statement-form` (the missing parser/compiler surface), `B-bpir-compile-property-vs-allocate-pins-ordering` (the subtle correctness rule for shape-determining UPROPERTYs), `E-bpir-classify-node-registry` (extension point for new K2Nodes), `E-bpir-struct-literal-formatter-duplicated` (cross-file resolver duplication), `E-bpir-magic-strings-and-duplications` (schema constants, dedup'd helpers, named constants for cross-file load-bearing literals). The decompiler-side `EmitGenericNode` (`BpirTextEmitter.cpp:1578-1598`) already does the pin-walked emit — the surface change required is on the compile side.
- `#4-residual-node-props-emit` `IN-REVIEW` developer — Appended `node_props { ... }` block to `EmitGenericNode` via `BuildSparsePropertyDiffJson` (CDO diff). New `BpirStructLiteralUtils::FormatJsonValueAsBpir` JSON→BPIR formatter routes struct values through the existing `ExportText` path. Kept `call` keyword (no `generic` introduction).
- `#5-misdiagnosis-supersedes-verify-fail` `IN-REVIEW` developer — A prior local re-open noted that live decompile of `K2Node_CreateWidget` invocations on `/App/App/UI/W_TrackObjectBrowser` and `/App/App/LevelBlueprints/B_DronePlayerController` emit no `node_props { ... }` block, framed as a missing emit. Three-agent investigation (engine pin-synthesis, plugin decompile/compile, round-trip contract) found this to be a misread of correct output, and the re-open was rolled back. Evidence: `UK2Node_CreateWidget` declares zero UPROPERTYs (verified `C:/UE_5.6/Engine/Source/Editor/UMGEditor/Private/Nodes/K2Node_CreateWidget.h:13`), and `UK2Node_ConstructObjectFromClass` / `UK2Node_SpawnActorFromClass` / `UK2Node_CallFunction` declare zero editable (`EditAnywhere|EditDefaultsOnly|BlueprintReadWrite|BlueprintReadOnly`) UPROPERTYs — so `BuildSparsePropertyDiffJson`'s `CPF_Edit \| CPF_BlueprintVisible` filter has nothing to surface for these node classes regardless of where in the emitter chain it runs. The bound widget class round-trips through the existing `Class:` pin arg, not a UPROPERTY. Synthesized expose-on-spawn pin values round-trip through `EmitGenericNode → FormatArgs` (where `K2Node_CreateWidget` actually dispatches — it derives from `UK2Node_ConstructObjectFromClass`, NOT `UK2Node_CallFunction`, so `GraphWalker::ClassifyNode` falls through to `ENodeSemantics::Unknown`); and back via `CodeNodeEmitter::CreateGenericK2Node` setting `ClassPin->DefaultObject` + `ReconstructNode()` then `WireDataPins` matching named args. Verified against cached `bpir.txt`: `W_TrackObjectBrowser` emits only `Class:`/`OwningPlayer:` because `W_TrackObjectBrowserPictureItem` declares zero `ExposeOnSpawn` properties (`asset-dumps/App/App/UI/LobbyAndMenu/Elements/W_TrackObjectBrowserPictureItem/properties.json:355-365`); `B_DronePlayerController` correctly emits `CheatText:` for the four CreateWidget call sites bound to `W_CheatsNotificator` (which has the lone `ExposeOnSpawn CheatText`); `UI_CameraList`/`UI_PopupSelector` emit 5–9 synthesized pins per call with struct literals. The pre-existing `#4` fix and its `FBpirCallK2NodeDecompileEmitsNodeProps` test on `K2Node_InputKey::bExecuteWhenPaused` (a real UPROPERTY) cover the residual scope correctly. No source change needed; status remains IN-REVIEW pending tester sign-off.
- `#6-verify-fix` `DONE` tester — Verified: live `blueprint.decompile` on `/App/App/LevelBlueprints/B_DronePlayerController` reproduces the negative-case evidence from `#5` exactly — four `K2Node_CreateWidget` calls bound to `W_CheatsNotificator` emit `Class:`/`CheatText:` args with no `node_props { }` block (correct, since `UK2Node_CreateWidget` and its `UK2Node_ConstructObjectFromClass` chain declare zero editable UPROPERTYs). Source review of `BpirTextEmitter.cpp:1651-1714` confirms `EmitGenericNode` builds the block via `BuildSparsePropertyDiffJson` + `BpirStructLiteralUtils::FormatJsonValueAsBpir`, sandwiches it between `(args)` and exec targets, and omits it when the diff is empty — matching the residual-scope contract. The positive case (a K2Node with a non-default editable UPROPERTY) is locked in by `FBpirCallK2NodeDecompileEmitsNodeProps` on `K2Node_InputKey::bExecuteWhenPaused`; the verifier protocol forbids running the automation suite, but the test exercises the changed code path with explicit `node_props {` and `bExecuteWhenPaused` assertions on the slice between braces.
