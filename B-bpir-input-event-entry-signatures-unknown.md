---
id: B-bpir-input-event-entry-signatures-unknown
title: "BPIR emits 'entry event UnknownEntry()' for InputAction/InputAxis/InputKey/InputTouch/ActorBound/GeneratedBound/WidgetAnimation events"
status: DONE
severity: High
category: bug
tags: [bpir, decompiler, entry-signature, input-events]
---

# Multiple K2Node entry types emit `UnknownEntry`

`EmitEntrySignature` (`Decompiler/BpirTextEmitter.cpp:497-537`)
handles only 5 entry-node K2Node types: `UK2Node_CustomEvent`,
`UK2Node_ComponentBoundEvent`, `UK2Node_Event`,
`UK2Node_FunctionEntry`, `UK2Node_InputKey`. Every other entry
node falls through to a final
`return TEXT("entry event UnknownEntry()")` branch.

**Affected node classes (per K2Node catalog audit):**
- `UK2Node_InputActionEvent` — Enhanced Input action triggers
- `UK2Node_InputAxisEvent` — legacy axis input
- `UK2Node_InputAxisKeyEvent` — axis input on a specific key
- `UK2Node_InputKeyEvent` — distinct from `UK2Node_InputKey`
  (which IS supported)
- `UK2Node_InputTouchEvent` — touch input
- `UK2Node_InputVectorAxisEvent` — vector-axis input
- `UK2Node_ActorBoundEvent` — actor-bound delegate event
- `UK2Node_GeneratedBoundEvent` — auto-generated bound event
- `UK2Node_WidgetAnimationEvent` — UMG animation-track event

**Why this is more severe than a body-node fallthrough:** the
entry signature *names* the function. `entry event UnknownEntry()`
is not just imprecise — it's globally ambiguous (multiple events
in the same BP collide on the same name) and defeats round-trip:
the compiler has no way to know which input action / axis / touch
to bind.

**Shape-determining properties** (per `B-bpir-compile-property-vs-allocate-pins-ordering`):
- `InputActionName` (`UK2Node_InputActionEvent`)
- `InputAxisName`, `Key` (`UK2Node_InputAxisEvent`,
  `UK2Node_InputAxisKeyEvent`)
- `InputKeyEvent`, `Key` (`UK2Node_InputKeyEvent`)
- `EventOwner`, `DelegatePropertyName` (`UK2Node_ActorBoundEvent`)
- `bExecuteWhenPaused` (most input nodes)
- `AnimationName`, `EventName` (`UK2Node_WidgetAnimationEvent`)

Each must be set before `AllocateDefaultPins()` for the entry
signature to materialize correctly.

## Fix scope

Two layers:

1. **Decompile (urgent):** add typed branches in `EmitEntrySignature`
   for each node class above, emitting a distinct entry-kind
   keyword: `entry input_action`, `entry input_axis`,
   `entry input_touch`, `entry input_key_event`,
   `entry actor_bound_event`, `entry widget_animation_event`,
   etc. Mirror the existing `UK2Node_InputKey` shape.
2. **Compile + parser:** extend `BpirParser::ParseEntryLine`
   (`BpirParser.cpp:575-619`) with the new entry-kind keywords,
   and add construction handlers in `BpirCompiler.cpp` that
   set the shape-determining properties before
   `AllocateDefaultPins()`.

Long-term, this is also a candidate for a `generic entry K2Node_<Type>(...)`
form analogous to `F-bpir-add-generic-node-statement-form`, since
the long tail of entry-event K2Nodes (especially Enhanced Input
plugin variants) is open-ended.

## Repro

Any BP using Enhanced Input or legacy InputAction/InputAxis events
in PDS. Cache root has examples in pawn / controller BPs.

## History
- `#1-initial-spec` `OPEN` reporter — K2Node catalog audit identified 8 entry-event node classes that fall through to `EmitEntrySignature`'s `UnknownEntry` branch. Distinct from body-node coverage gap (`B-bpir-decompiler-emitter-coverage-gap`) because entry signatures name the function and `UnknownEntry` collides on multiple events. Required for any BP using Enhanced Input or UMG animation events to round-trip.
- `#2-typed-entry-emit` `IN-REVIEW` developer — Added typed entry-signature branches in `BpirTextEmitter.cpp::EmitEntrySignature` for the 8 input/bound/widget-animation entry classes. Each emits a distinct keyword form (e.g. `entry input_action_event Jump(IE_Pressed)`) using the existing `AppendEntryPosition`/`CollectParams` infrastructure. Decompile-only fix per ticket scope; parser/compiler is a deferred layer-2 follow-up. Test `FDecompilerInputActionEventEntrySignatureTest` covers the `UK2Node_InputActionEvent` case via `FBpirDecompiler::Decompile()`.
- `#3-verify-input-entry-decompile` `DONE` tester — Verified: `blueprint.decompile` on `/Game/Blueprints/Pawn/PW_Crane` emitted `entry input_axis MoveForward` / `entry input_axis MoveRight` and no `UnknownEntry`; sampled `/Game/Blueprints/Pawn/ThirdPersonCharacter` also emitted `entry input_axis Turn/MoveForward/LookUp/MoveRight` plus input action entries with no `UnknownEntry`.
