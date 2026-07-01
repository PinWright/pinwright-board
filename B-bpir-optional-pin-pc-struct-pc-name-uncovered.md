---
id: B-bpir-optional-pin-pc-struct-pc-name-uncovered
title: "BPIR decompiler emits 'Key: ?' for FKey pins wired to UK2Node_InputKey entry"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, input-key, fkey, wire-resolver]
---

# `Key: ?` emitted for wires sourced from a UK2Node_InputKey entry node

`#2-pc-struct-pc-name-omittable` / `#4-fkey-empty-struct-omission` /
`#6-call-function-struct-fallback` extended `IsPinOmittableAtCallSite`
in `BpirTextEmitter.cpp` for empty/default-equivalent PC_Struct + PC_Name
pins. Those changes still ship, but they were aimed at the wrong code
path for the returned repro.

- `Game\ObjectiveManager\Components\BPC_Objective\bpir.txt` —
  `WasInputKeyJustPressed(Target: %n1, Key: ?)`

## Why the existing predicate still misses FKey

The repro's `Key` pin **is wired**, not unwired. Live introspection of
`/Game/ObjectiveManager/Components/BPC_Objective` confirms the chain
`UK2Node_InputKey "E".Key → UK2Node_Knot.OutputPin → WasInputKeyJustPressed.Key`.
`IsPinOmittableAtCallSite` returns false at the
`Pin->LinkedTo.Num() != 0` guard before any of the PC_Struct / PC_Name
short-circuit branches run, so the omit path was never the right lever.

The bug actually lives in the decompiler's wire resolver
(`BpirDecompiler.cpp::ResolveInputValue`). The handler list covers
`UK2Node_Self`, `UK2Node_CustomEvent`, `UK2Node_Event`,
`UK2Node_FunctionEntry`, the macro-entry tunnel, and variable get/set —
but not `UK2Node_InputKey`. A `UK2Node_InputKey` is treated as an entry
node by the emitter and never gets a `NodeToValueName` entry, so the
wire-resolver falls through to the "visited but produced no value name"
branch and returns `?`. The decompile output already contains the
matching warning: `"Unresolvable value: source node 'E' was visited but
produced no value name"`. `UK2Node_InputAction` (also exposes a `Key`
FKey output pin) has the same defect, but the production node has no
constant FKey member to render — only the wired-to-InputKey-entry shape
is fixable as an inline literal.

The fix is to add a `UK2Node_InputKey` branch in `ResolveInputValue`
that returns the inline FKey literal (`InputKey.GetDisplayName()` with
whitespace stripped, matching the entry-signature emit transform in
`BpirTextEmitter.cpp`) when the source pin is the entry's `Key` data
output. The PC_Struct / PC_Name omit-predicate work in the emitter is
left in place — it covers genuinely unwired optional pins on other
call sites.

## History
- `#1-initial-repro` `OPEN` reporter — Re-audit of 2026-05-04 cache + sentinel sweep found 8+ `: ?` survivors on `OptionalName: FName` (CreateDynamicMaterialInstance, 7 cases) and `Key: FKey` (WasInputKeyJustPressed, 1+ cases). Follow-up to `B-bpir-optional-pin-no-default-question-mark` (DONE); `IsPinOmittableAtCallSite` short-circuits do not cover PC_Struct or PC_Name. Recommended generalization: detect default-equivalence via `UScriptStruct::CompareScriptStruct` for PC_Struct, treat empty/None for PC_Name as omittable.
- `#2-pc-struct-pc-name-omittable` `IN-REVIEW` developer — Extended `IsPinOmittableAtCallSite` with PC_Name (empty/"None") and PC_Struct (default-equality via `FStructOnScope`+`UScriptStruct::CompareScriptStruct`) short-circuit branches in `BpirTextEmitter.cpp`. Generalizes the existing PC_Object pattern; FKey, FName, and any struct-typed optional pin now omit cleanly when at default. Test `FDecompilerOmitsOptionalStructNameUnwiredPinsTest` covers `WasInputKeyJustPressed.Key` and `CreateDynamicMaterialInstance.OptionalName`.
- `#3-returned-fkey-still-emits` `OPEN` tester — Returned: `/Game/ObjectiveManager/Components/BPC_Objective` still decompiles `WasInputKeyJustPressed(Target: %n1, Key: ?)`, while `/Game/CarConfigurator/Shared/UI/UI_CameraButton` no longer emits `OptionalName: ?`. Test: `blueprint.decompile` on both repro assets.
- `#4-fkey-empty-struct-omission` `IN-REVIEW` developer — Narrowed the returned optional-pin issue to the remaining FKey/PC_Struct repro, moved empty struct default omission before the resolved UScriptStruct requirement, and strengthened FDecompilerOmitsOptionalStructNameUnwiredPinsTest with the loaded-asset FKey shape.
- `#5-fkey-still-emits` `OPEN` tester — Returned: `blueprint.decompile` on `/Game/ObjectiveManager/Components/BPC_Objective` still emits `WasInputKeyJustPressed(Target: %n1, Key: ?)`. Empty `PC_Struct` fast path either didn't ship in the running editor build or doesn't trigger on this loaded FKey pin shape. Test: `mcp__editor-automation__call("blueprint.decompile", {"assetPath": "/Game/ObjectiveManager/Components/BPC_Objective"})`.
- `#6-call-function-struct-fallback` `IN-REVIEW` developer — Added call-function parameter struct fallback for optional PC_Struct pins whose PinSubCategoryObject is missing, so default-equivalent FKey values can still be omitted instead of emitted as `Key: ?`; strengthened `FDecompilerOmitsOptionalStructNameUnwiredPinsTest` to clear the FKey pin struct object and assert the fallback path.
- `#7-returned-fkey-still-emits` `OPEN` tester — Returned: `blueprint.decompile` on `/Game/ObjectiveManager/Components/BPC_Objective` still emits `WasInputKeyJustPressed(Target: %n1, Key: ?)`, so the call-function struct fallback did not cover the loaded FKey pin shape in the running editor. Test: `mcp__editor-automation__call("blueprint.decompile", {"assetPath": "/Game/ObjectiveManager/Components/BPC_Objective"})`.
- `#8-decompiler-input-key-source` `IN-REVIEW` developer — Diagnosis was wrong on prior rounds: the FKey pin in BPC_Objective is wired through a knot to a K2Node_InputKey 'E', so IsPinOmittableAtCallSite is bypassed entirely. Added UK2Node_InputKey/UK2Node_InputAction branch in BpirDecompiler::ResolveInputValue to emit the inline FKey literal. FDecompilerInputKeyConstOutputResolvesAsLiteralTest covers the wired-via-knot shape; the prior PC_Struct/PC_Name omit-predicate work in BpirTextEmitter is left in place.
- `#9-verify-fkey-inline-literal` `DONE` tester — Verified: `blueprint.decompile` on `/Game/ObjectiveManager/Components/BPC_Objective` now emits `call WasInputKeyJustPressed(Target: %n1, Key: E)` with the inline FKey literal; no `Key: ?` and no "Unresolvable value: source node 'E'" warning in the decompile output.
