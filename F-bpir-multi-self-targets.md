---
id: F-bpir-multi-self-targets
title: "Multi-self fan-out emits a mixed authored-position body that fails recompile"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, decompiler, self-pin, fan-out, multi-target, authored-position, round-trip]
encounters: 6
lastSeen: 2026-06-27T19:02:58Z
---

# Multi-self fan-out emits a mixed authored-position body that fails recompile

> Reframed from the original "loses fan-out targets" report. The per-target
> fan-out feature it described is SHIPPED and verified (history #2–#4); the live
> open defect is the #5/#6 positioned-multiself regression captured below.

## Why

The decompiler fans out a multi-self node — one `UK2Node_CallFunction` with
`AllowMultipleSelfs(false) == true`, or a `UK2Node_BaseMCDelegate`-derived
dispatcher node, whose `self` pin has >1 link (`NodeSupportsMultiSelf`) — into N
sibling `call`/dispatcher lines, one per linked target, joined by `\n`
(`FBpirTextEmitter::EmitCallNode` ~line 1529, `EmitDispatcherNode` ~line 1763;
`FormatTargetPrefix` swap-promotes each `LinkedTo` entry to slot 0 to resolve
all sources). The fan-out itself is correct and shipped: UE compiles a multi-self
call into N independent runtime invocations, so emitting N siblings preserves
semantics, and the old "only the first is used" warning is suppressed for this
case.

The defect is how those siblings are positioned. `FBpirDecompiler::AppendNodeLine`
(`BpirDecompiler.cpp:308-320`) receives the whole `\n`-joined block as ONE `Line`
and wraps it with a single leading 4-space indent and a single trailing `@(x, y)`:

    Printf("    %s @(%d, %d)", Line, NodePosX, NodePosY)

So only the FIRST physical sibling gets the body indent and only the LAST sibling
gets the node's `@(x, y)`; the in-between siblings sit at column 0 with no
position. Because all N siblings are emissions of ONE graph node sharing ONE
position, this lands the entry body in *mixed* authored-position mode, which the
compiler forbids ("every visible node-backed instruction in one entry body must
either all have @(x, y) or none do", `BpirCompiler.cpp:1896`). The decompiler's
own verbatim output therefore no longer recompiles — `compile_bpir` returns
`COMPILE_FAILED` and rolls back cleanly.

This is a regression of the history #2 per-target fan-out fix: its claim
"round-trips through the parser unchanged" holds only for the unpositioned
(auto-layout) case; for any positioned multi-self node it converted a silent
dropped-target warning into a hard recompile failure. The existing
`FBpirRoundTripMultiSelfTargetsTest` missed it because it only counts emitted
`call` occurrences and never feeds the decompile back into the compiler.

Replay-confirmed live (zero mutation, stock content): function body
`/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`
"Update light color" (one `Set Vector Parameter Value` at (512, -16) with two
self links) and ubergraph event body
`/Game/ExampleContent/Blueprints/Blueprints/BP_Gears` EventGraph Tick (one
"Add Local Rotation" at (1888, 192) with two self links). In both, decompile
emits the first sibling indented/unpositioned and the last sibling at column 0
carrying the node's only `@(x, y)`, and feeding that verbatim back into
`compile_bpir` fails with the mixed-authored-position error.

## Fix

In `FBpirDecompiler::AppendNodeLine`, split the incoming `Line` on `\n` and apply
the 4-space body indent and the node's `@(x, y)` to EVERY physical sibling line —
all siblings emit the same graph node and share its one position, so never emit a
partially-positioned sibling set. Map the node to its FIRST sibling line in
`NodeToLineIndex` (reconvergence's retroactive label insertion targets the start
of the group). This is the single choke point: both `EmitCallNode` and
`EmitDispatcherNode` route their `\n`-joined fan-out blocks through
`AppendNodeLine`. No emitter or compiler change is needed.

Extend `FBpirRoundTripMultiSelfTargetsTest` (which already authors the call at
(200, 0)) to assert every sibling `call` line is uniformly indented and carries
`@(200, 0)`, and that the decompiled text recompiles onto a fresh Blueprint with
no mixed-authored-position `COMPILE_FAILED`.

## Cross-references

- `F-bpir-multi-input-exec-targets` — DONE — exec-side prior art
  (fan-in on input exec pins via `@label.PinName` suffix).
- `B-bpir-create-event-self-pin-unresolved` — adjacent self-pin
  resolution gap.
- `E-bind-dispatcher-redundant-self-wire` — another self-pin emission
  quirk.

## History
- `#1-filed-from-music-subsystem-bug` `OPEN` reporter — Discovered while debugging a PDS music-context bug (`W_HUD_Race_Racing` calls `OnRaceStarted` three times via fan-in on the self pin; decompile only shows one call plus a warning). Verified against engine source that UE compiles multi-self into N runtime calls for impure no-return functions, so the asset is correct and the decompiler is dropping semantics.
- `#2-per-target-fan-out` `IN-REVIEW` developer — Implemented approach 1 (per-target line expansion). `BpirTextEmitter::FormatTargetPrefix` now returns `TArray<FString>`; `EmitCallNode` and `EmitDispatcherNode` emit one sibling `call FuncName(Target: ...)` line per linked source when `NodeSupportsMultiSelf` (UK2Node_CallFunction with `AllowMultipleSelfs(false) == true` or UK2Node_BaseMCDelegate-derived) and no result/value is consumed. `BpirDecompiler::ResolveInputValue` suppresses the "only the first is used" warning for the multi-self case. Regression test `FBpirRoundTripMultiSelfTargetsTest` covers both the per-target emission and the warning suppression.
- `#3-verify-fix` `DONE` tester — Verified: ran `system.run_tests` with `test=EditorAutomationRpcGateway.bpir.roundtrip.MultiSelfTargets`; PDS.log shows `Test Completed. Result={Success}` for that test. The regression test directly asserts three sibling `call SetActorHiddenInGame(Target:` lines and suppression of the "only the first is used" warning. Cross-checked with live decompile of `/App/App/UI/LobbyAndMenu/HUD/W_HUD_Race_Racing` — `warnings: []` (vs the cached pre-fix `bpir.txt` which still carries `BPIR_FAILED: Data pin 'self' has 3 connections but only the first is used`); the live `OnRaceStarted.self` pin now has only 1 LinkedTo (graph repaired), so the single emitted call there is correct.
- `#4-reverify-fix` `DONE` tester — Re-verified: ran `system.run_tests` with `test=EditorAutomationRpcGateway.bpir.roundtrip.MultiSelfTargets` via job ticket `j_20260521T050007_8a5adab6`; status `completed`, `has_errors: false`, `resolvedTests` matches `requestedTests`, `missingTests: []`. The regression test asserts both per-target emission and warning suppression; passing it live confirms the multi-self fan-out fix holds.
- `#5-regression-positioned-multiself-not-recompilable` `OPEN` reporter — Regression: the #2 per-target line-expansion fix breaks decompile→compile round-trip whenever the multi-self node's body is in **authored-position mode**. The emitter expands one multi-self `K2Node_CallFunction` into N sibling `call` lines but attaches the node's single `@(x, y)` to only ONE sibling, leaving the other N-1 position-less. All N siblings map to one graph node with one position, so this lands the entry body in *mixed* authored-position mode — which `compile_bpir` forbids (`blueprint.compile_bpir` wiki line 22) — and the decompiler's own verbatim output no longer recompiles. The `#2` claim "round-trips through the parser unchanged because each emitted call is a normal single-target call" holds only for the unpositioned (auto-layout) case; the bug merely changed shape (silent dropped-targets → hard `COMPILE_FAILED`), it did not close the round-trip for positioned bodies. The existing `FBpirRoundTripMultiSelfTargetsTest` missed it because it does not author positions on the multi-self call. **Replay-confirmed live this session (zero mutation, stock asset)** on `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`: the `Update light color` function has exactly ONE `Set Vector Parameter Value` node (`blueprint.graph.get_nodes`: nodeId `2E16AC4B4D486071BD4CFEB9C7F14A5B`, `x:512 y:-16`, whose `self` input pin `linkedTo` BOTH `Get Light bulb material 1` and `Get Light bulb material 2`). `blueprint.decompile` emits it as two sibling lines, the first with no position and the second carrying the node's only `@(512, -16)` — and the second sibling also loses its body indentation (emitted at column 0):

```
    call SetVectorParameterValue(Target: $`Light bulb material 1`, ParameterName: "Emissive color", Value: $`New color`)
call SetVectorParameterValue(Target: $`Light bulb material 2`, ParameterName: "Emissive color", Value: $`New color`) @(512, -16)
```

Feeding that verbatim decompile back into `blueprint.compile_bpir` (`mode: "append"`) fails: `[COMPILE_FAILED] ... Line 29: Manual BPIR placement error in entry body 'Update light color' at source line 29 (SetVectorParameterValue): mixed authored-position mode is forbidden because every visible node-backed instruction in one entry body must either all have @(x, y) or none do.` Failed compile rolled back cleanly (re-decompile byte-identical, editor responsive). **Fix:** in the per-target expansion (`BpirTextEmitter::FormatTargetPrefix` / `EmitCallNode`), replicate the source node's `@(x, y)` onto EVERY sibling line (and preserve body indentation on every sibling), since all siblings are emissions of the *same* graph node and share its one position; never emit a partially-positioned sibling set. Extend `FBpirRoundTripMultiSelfTargetsTest` to author a position on the multi-self call and assert decompile→compile→decompile round-trips. NOTE (separate defect, NOT this ticket): the same asset's `Toggle light` event also fails to round-trip on `Line 11: Could not resolve value '%n1.flash' for pin 'A'` — the decompiler references a `timeline` node's track output pin (`%n1.flash`) that the BPIR `timeline NAME()` opcode does not recreate (timeline-track modeling gap, cf. `F-graph-create-node-timeline`); tracked elsewhere, mentioned here only as round-trip context.
- `#6-additional-evidence-bp-gears-event-body` `OPEN` reporter — Additional evidence (independent replay, zero-mutation stock content, second distinct asset): the #5 positioned-multiself defect also reproduces in an **ubergraph event** body, not just a function body. On `/Game/ExampleContent/Blueprints/Blueprints/BP_Gears` `EventGraph`, `blueprint.graph.get_nodes` shows exactly ONE `K2Node_CallFunction` "Add Local Rotation" node (`nodeName: K2Node_CallFunction_35975`, `x:1888 y:192`) whose `self` input pin `linkedTo` BOTH `Get Gear 2` and `Get Gear 3`. `blueprint.decompile` fans that single node into two sibling lines inside the `entry event Tick(...) @(-96, 208)` body — the first indented with NO `@(x, y)`, the second at column 0 carrying the node's only `@(1888, 192)`:

```
    call K2_AddLocalRotation(Target: $`Gear 2`, DeltaRotation: %n5, bSweep: false, bTeleport: false)
call K2_AddLocalRotation(Target: $`Gear 3`, DeltaRotation: %n5, bSweep: false, bTeleport: false) @(1888, 192)
```

Every other node-backed instruction in the Tick body carries `@(x, y)`, so the un-positioned Gear 2 sibling is the mixed-authored-position violation `compile_bpir` rejects — the body is non-recompilable, exactly the #5 shape. Confirms the regression is not asset-specific and affects the very common ubergraph-event surface. (Did not re-run the round-trip compile here to avoid the documented COMPILE_FAILED→undo-broadcast editor-hang risk; the verbatim decompile + single-node get_nodes cross-check is conclusive for the decompiler-side emission defect.)
- `#7-reword-and-fix-positioned-multiself` `IN-REVIEW` developer — Reworded the ticket: the title/Why/Proposed-fix described the already-shipped (#2–#4) per-target fan-out feature; reframed to the live #5/#6 positioned-multiself regression. Root-cause fix at the single choke point `FBpirDecompiler::AppendNodeLine` (`BpirDecompiler.cpp:308`): it now splits the incoming `\n`-joined emission into physical lines and applies the 4-space body indent AND the node's `@(x, y)` to EVERY sibling line (was: one indent on the first sibling, one `@(x, y)` on the last, the rest at column 0 → mixed authored-position mode the compiler rejects), and maps the node to its FIRST sibling line in `NodeToLineIndex` (so reconvergence label insertion targets the group start). This covers both `EmitCallNode` and `EmitDispatcherNode`, which both route `\n`-joined fan-out blocks through `AppendNodeLine`; no emitter/compiler change. Regression test: extended `FBpirRoundTripMultiSelfTargetsTest` (`TestBpirRoundTrip.cpp`, the call is authored at (200, 0)) to assert all three sibling `call SetActorHiddenInGame(Target:` lines are uniformly indented and carry `@(200, 0)`, and that the decompiled text recompiles onto a fresh Blueprint carrying the same three actor variables (no mixed-authored-position `COMPILE_FAILED`). Both assertions fail if the AppendNodeLine fix is reverted. Files: `Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`, `Source/PinWright/Private/Tests/Bpir/TestBpirRoundTrip.cpp`.
