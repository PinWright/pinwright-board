---
id: F-bpir-field-notify-primitive
title: "Add BPIR primitive for FieldNotification subscription"
status: DONE
severity: Medium
category: feature
tags: [bpir, field-notification, k2-add-field-value-changed-delegate, create-delegate, primitive]
---

# Add BPIR primitive for FieldNotification subscription

BPIR has no first-class way to emit the canonical UE 5.4+ FieldNotification
subscription pattern (`K2_Add/RemoveFieldValueChangedDelegate` + paired
`UK2Node_CreateDelegate` wired to a local event). Authoring it as a generic
`call` with `Delegate: $OutputDelegate` fails to compile with
`TryCreateConnection failed wiring data 'OutputDelegate' -> 'Delegate'`
because the compiler does not synthesize the supporting
`UK2Node_CreateDelegate` node — there is no decompiler emission contract
that the compiler would recognise (investigated during the sprint; the
`$OutputDelegate` form is an accidental rendering from the generic
pure-node fallback, not an intentional round-trip shape).

The closest prior art is `bind_dispatcher`, which IS a first-class opcode
whose compiler synthesizes the `UK2Node_CreateDelegate` + self/target pin
wiring + `SelectedFunctionName` + `HandleAnyChange()`
(`BpirCompiler.cpp:4175-4395` + matching decompiler emitter in
`BpirTextEmitter.cpp:691-772`). FieldNotification needs the same treatment,
just with `K2_Add/RemoveFieldValueChangedDelegate` as the parent node and
a `FieldName` alongside `target` and `event`.

## Proposed syntax

```
field_notify_subscribe <FieldName>(target: self, event: @Handler)
field_notify_unsubscribe <FieldName>(target: self, event: @Handler)
```

- `target:` defaults to `self`; external targets allowed (anything
  implementing `INotifyFieldValueChanged`, which includes `UUserWidget`).
- `event:` is a `@LocalEvent` (custom event or function) with signature
  matching `FFieldValueChangedDynamicDelegate(UObject* Object,
  FFieldNotificationId Field)` — note single-cast (`PC_Delegate`), not
  multicast (`PC_MCDelegate`).

## UE API surface (confirmed during investigation)

- `UWidget::K2_AddFieldValueChangedDelegate` / `K2_RemoveFieldValueChangedDelegate`
  — `Engine/Source/Runtime/UMG/Public/Components/Widget.h:786,789`. Plain
  `UFUNCTION(BlueprintCallable)` methods, so the parent is a
  `UK2Node_CallFunction`.
- `FFieldValueChangedDynamicDelegate` — single-cast dynamic delegate in
  `Engine/Source/Runtime/FieldNotification/Public/INotifyFieldValueChanged.h:12`.
- `FFieldNotificationId` struct with one `FName FieldName` field
  (`Engine/Source/Runtime/FieldNotification/Public/FieldNotificationId.h:125`).
  There is **no** separate `UK2Node_MakeFieldValueChangedDelegate` — the
  struct literal `(FieldName="X")` is the whole input for the `FieldId` pin.
- `UK2Node_CreateDelegate` — `Engine/Source/Editor/BlueprintGraph/Private/K2Node_CreateDelegate.cpp`.
  Output pin `OutputDelegate` (:36), input pin `Self` (:37,63). Methods:
  `CreateNewGuid`, `PostPlacedNewNode`, `AllocateDefaultPins`,
  `GetDelegateOutPin`, `GetObjectInPin`, `SetFunction(FName)`,
  `HandleAnyChange()`.

## Implementation surface (sketch)

1. `BpirTypes.h` (~line 40): add `FieldNotifySubscribe` and
   `FieldNotifyUnsubscribe` to `EBpirOpcode`; update the
   `IsTerminator`/printer switch (~:99).
2. `BpirTokenizer.cpp:36-37`: add the two keywords to the keyword list.
3. `BpirParser.cpp:~848`: add parse handlers accepting a bareword
   `FieldName` literal plus `target:`/`event:` args — wrap or thin-adapt
   `ParseDispatcherInstruction`.
4. `BpirCompiler.cpp:4175-4395`: extend the dispatcher opcode block.
   Generalise `CreateDelegateNode` to accept either a
   `UK2Node_BaseMCDelegate` template or a `UK2Node_CallFunction` targeting
   `K2_Add/RemoveFieldValueChangedDelegate`. For field-notify:
   - Build the `UK2Node_CallFunction`; set the `FieldId` struct-literal
     pin default to the parsed `FieldName`.
   - Reuse the existing `UK2Node_CreateDelegate` synthesis block verbatim
     (:4296-4357) to wire the `Delegate` input pin.
5. `BpirCompiler.cpp:~5432`: add the opcodes to the purity/validation
   switch.
6. `BpirSubgraphCompiler.cpp:32-33`: register the new keywords alongside
   `bind_dispatcher`.
7. `BpirTextEmitter.cpp:~691`: add an emitter matched on parent =
   `UK2Node_CallFunction` with target function
   `K2_Add/RemoveFieldValueChangedDelegate`. Extract `FieldId.FieldName`
   from the struct default, walk the `Delegate` pin's `LinkedTo` for a
   `UK2Node_CreateDelegate` (same logic as dispatcher emitter at :740-771),
   and emit `field_notify_subscribe FieldName(target: ..., event: @Fn)`.
8. Decompiler dispatch: add the field-notify function-name branch
   wherever the dispatcher emitter is currently selected.
9. `docs/bpir-language-reference.md`: document under the dispatcher family.

## Regression test

`Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirFieldNotifySubscribeIntegrity.cpp`, modelled after
`TestBpirBindDispatcherExternalLocalEventIntegrity.cpp`:
- Compile BPIR that uses both opcodes against a widget BP with a
  `FieldNotify` property and a local `SetValues(UObject*, FFieldNotificationId)`
  event; assert compile succeeds with zero errors.
- Walk the graph and assert exactly one `K2_AddFieldValueChangedDelegate`
  call node and one `K2_RemoveFieldValueChangedDelegate` call node, each
  with `FieldId.FieldName == "Replay"` struct default and a
  `UK2Node_CreateDelegate` upstream whose `SelectedFunctionName == "SetValues"`.
- Negative test: the generic-`call`-with-`$OutputDelegate` form still
  fails — guards against an overeager fallback.

## Impact

Any BPIR-driven port of a UMG BP that uses reactive field bindings (the
canonical UE 5.4+ pattern) currently must skip the subscription and push
state explicitly from the spawn site — losing reactive updates.

Related but distinct from `B-bpir-bind-dispatcher-external-target-local-event`
(multicast dispatcher; different node class).

## History
- `#1-reclassified-from-b-ticket` `OPEN` reporter — Split out from
  `B-bpir-field-notification-delegate-pin-wire-fails` during mcp-sprint
  analysis. The original ticket framed this as a decompiler/compiler
  round-trip asymmetry bug, but investigation confirmed no prior
  emission contract existed for `$OutputDelegate` — it was an accidental
  rendering from the generic pure-node fallback. The correct fix is a
  new primitive analogous to `bind_dispatcher`, making this a feature
  request. Original repro (W_RenameReplay Construct/Destruct) carries
  over.
- `#2-bpir-field-notify-opcodes` `IN-REVIEW` developer — Added FieldNotifySubscribe/FieldNotifyUnsubscribe BPIR opcodes mirroring bind_dispatcher. Compiler synthesizes UK2Node_CallFunction for K2_Add/RemoveFieldValueChangedDelegate with FieldId pin default and a UK2Node_CreateDelegate wired to a local event handler. Decompiler reuses ENodeSemantics::Dispatcher and extends EmitDispatcherNode keyword chain. Refactored shared CreateDelegate-wiring tail into WireCreateDelegateForBindNode helper. Added regression test FBpirFieldNotifySubscribeIntegrityTest.
- `#3-verified-roundtrip` `DONE` tester — Verified live on a temp widget BP (`/Game/App/UI/Test/W_McpVerifyTemp`): compiled BPIR `entry custom_event SetValues(object<Object> InObject, struct<FieldNotificationId> InField){} entry event Construct(){ field_notify_subscribe Replay(target: self, event: @SetValues) }` → `compiled:true, status:UpToDate, errors:[], nodeCount:5` with 5 created GUIDs. Decompile round-trip returned `field_notify_subscribe Replay(event: @SetValues)` with no warnings. Parser, compiler, and decompiler emitter all wired through.
