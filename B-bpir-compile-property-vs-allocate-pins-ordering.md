---
id: B-bpir-compile-property-vs-allocate-pins-ordering
title: "Generic K2Node compile path must enforce property-vs-AllocateDefaultPins ordering for shape-determining properties"
status: DONE
severity: High
category: bug
tags: [bpir, compiler, allocate-default-pins, ordering]
---

# Generic compile path must respect property-vs-pins ordering

The audit of `BpirCompiler.cpp` identified the most subtle
correctness rule for the planned generic K2Node compile path
(`F-bpir-add-generic-node-statement-form`): some K2Node UPROPERTYs
must be set **before** `AllocateDefaultPins()` because the engine
reads them at allocation time to derive the pin shape; others can
be set after. A naive "construct node → AllocateDefaultPins → replay
all node_props" implementation will silently produce wrong pin sets
for these classes.

## Property-before-AllocateDefaultPins (mandatory)

| Class | Property / call | Source |
|-------|-----------------|--------|
| `UK2Node_Event` | `EventReference.SetExternalMember(...)` + `bOverrideFunction = true` | `BpirCompiler.cpp:3279-3282` |
| `UK2Node_CallDelegate` / `AddDelegate` / `RemoveDelegate` / `ClearDelegate` | `SetFromProperty(DelegateProp, bSelfContext, SearchClass)` | `BpirCompiler.cpp:4770` |
| `UK2Node_CallFunction` (FieldNotify forms) | `SetFromFunction(TargetFunc)` | `BpirCompiler.cpp:4974-4975` |
| `UK2Node_FormatText` | format string drives dynamic pin generation at construction | `BpirCompiler.cpp:3700` |
| `UK2Node_AsyncAction` | `InitializeProxyFromFunction(FactoryFn)` before AllocateDefaultPins | `CodeNodeEmitter.cpp:985-990` |
| `UK2Node_ConstructObjectFromClass` (and subclasses incl. `UK2Node_SpawnActorFromClass`, `UK2Node_CreateWidget`) | Class pin must be populated before `ReconstructNode()` to derive typed output pins | `CodeNodeEmitter.cpp:1003-1020` |

## Pin-mutation between AllocateDefaultPins and ReconstructNode

| Class | Action | Source |
|-------|--------|--------|
| `UK2Node_FunctionEntry` / `UK2Node_FunctionResult` / `UK2Node_CustomEvent` / `UK2Node_Tunnel` | `CreateUserDefinedPin(...)` for each parameter, then `ReconstructNode()` | `BpirCompiler.cpp:2856, 3463, 3482, 3499, 3635, 3636` |
| `UK2Node_SwitchName` / `SwitchInteger` / `SwitchString` | `CreatePin(EGPD_Output, PC_Exec, CaseName)` per case + maintain `Node->PinNames` array | `BpirCompiler.cpp:4272-4316` |

## Post-wire hooks

| Class | Action | Source |
|-------|--------|--------|
| `UK2Node_CreateDelegate` | `CDNode->SetFunction(FName)` then **after** the object pin is wired, `CDNode->HandleAnyChange()` | `BpirCompiler.cpp:5103-5104` |

## Properties safe to replay after AllocateDefaultPins

Every other UPROPERTY on a K2Node — boolean flags, references that
don't mutate the pin set, layout hints. These are the "ordinary"
case the generic path can handle by FProperty reflection without
ordering hazard.

## What the generic compile path needs

The `F-bpir-add-generic-node-statement-form` ticket sketches the
compile sequence. To respect the ordering above, the generic path
must:

1. Maintain a `TSet<FProperty*>` (or per-class metadata) of
   "shape-determining" properties for the K2Node class. Default to
   "no shape-determining properties" for unknown classes — they
   land in the safe post-Allocate branch.
2. Replay shape-determining `node_props` entries **before**
   `AllocateDefaultPins()`.
3. Replay the remaining `node_props` entries **after**
   `AllocateDefaultPins()`.
4. For any class touched by the post-Allocate replay, run
   `ReconstructNode()` afterwards.
5. Defer `HandleAnyChange()`-style post-wire hooks until after pin
   wiring completes; expose them through a per-class hook table.

The shape-determining-property metadata can start as a hardcoded
table for the six classes above and be expanded as new cases
surface. Long-term, classes that require this ordering should
ideally annotate their UPROPERTYs with a metadata tag
(e.g. `meta = (BpirShapeDetermining)`), but that requires engine
changes and is out of scope.

## Why high severity

Without this, the generic compile path silently produces broken
nodes for the most common dynamic-pin K2Node families (Event,
delegate, FormatText, SpawnActor, CreateWidget, AsyncAction). The
generic round-trip story collapses on exactly the cases the
umbrella issue was filed to address.

## History
- `#1-initial-spec` `OPEN` reporter — Audit of `BpirCompiler.cpp` enumerated 6 classes requiring property-before-AllocateDefaultPins, 4 requiring pin-mutation-before-ReconstructNode, and 1 requiring post-wire HandleAnyChange. Generic compile path (`F-bpir-add-generic-node-statement-form`) must respect these or silently produce wrong pin sets.
- `#2-shape-metadata-implemented` `IN-REVIEW` developer — Added `BpirShapeMetadata.{h,cpp}` with `FindBpirShapeDescriptor` (walks superclass chain) covering 6 K2Node families plus `RunPostWireHooks` for `UK2Node_CreateDelegate::HandleAnyChange`. `ReplayGenericNodeProps` partitions `node_props` into pre-Allocate / post-Allocate, replays in order, calls `ReconstructNode` if post-Allocate properties touched. Replaces Task 1's no-op stubs.
- `#3-skip-editor-offline` `SKIP` tester — Cannot exercise via MCP: Unreal Editor not running (port 19880 unreachable, no UnrealEditor.exe process). Verified source artifacts present: `Source/EditorAutomationRpcGateway/Private/Compiler/BpirShapeMetadata.{h,cpp}` exposes `FindBpirShapeDescriptor`/`ReplayGenericNodeProps`/`RunPostWireHooks`, and `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirCallK2NodeShapeOrdering.cpp` provides a FormatText-based pin-shape regression test, but runtime confirmation requires the editor.
- `#4-verify-formattext-shape` `DONE` tester — Verified live: created temp BP `/Game/App/UI/Test/W_McpVerifyTemp_B-bpir-compile-property-vs-allocate-pins-ordering`, ran `blueprint.compile_bpir` with `entry function TestFn() { %r = call K2Node_FormatText() node_props { Format: NSLOCTEXT("Test","Hello","Hello {Name}, you have {Count} items") } }`, then `blueprint.graph.get_nodes` showed the resulting `K2Node_FormatText_0` carrying both dynamic wildcard pins `Name` and `Count` alongside the default `Format`/`Result` pins — proves `Format` was replayed before `AllocateDefaultPins`/`PinDefaultValueChanged` so `PinNames` populated and `ReconstructNode` materialized the argument pins. Counterfactual (post-Allocate replay) would have left only `Format`/`Result`.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
