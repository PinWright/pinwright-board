---
id: E-bpir-createwidget-pin-hint
title: "`K2Node_CreateWidget` pin-name errors should hint the correct `Class:` name"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, k2node-createwidget, pin-name, error-hint]
---

# `K2Node_CreateWidget` pin-name errors should hint the correct `Class:` name

The input pin on `K2Node_CreateWidget` that accepts the widget subclass is named `Class`. Authors frequently guess `WidgetType` or `WidgetClass` (the latter is the name on `K2Node_AsyncAction_PushContentToLayerForPlayer`'s equivalent pin — a reasonable mistake given the name similarity).

Using the wrong pin name produces this error:

```
COMPILE_FAILED: Line N: Could not find target pin 'WidgetType' on node 'Create Widget'
```

The error correctly identifies the problem but doesn't tell you the correct pin name. Authors have to decompile a reference BP that uses CreateWidget to discover `Class` is the pin. Minor friction, but it hits on the first attempt virtually every time an author writes a new CreateWidget call.

## Repro (observed this session)

```
%item = call K2Node_CreateWidget(Class: /App/.../W_MyReplayListItem_C, OwningPlayer: %player)       # works
%item = call K2Node_CreateWidget(WidgetType: /App/.../W_MyReplayListItem_C, OwningPlayer: %player)  # fails
%item = call K2Node_CreateWidget(WidgetClass: /App/.../W_MyReplayListItem_C, OwningPlayer: %player) # fails
```

**Proposed fix:** When the BPIR compiler rejects a pin name, include a "did you mean" hint listing the closest-matching actual pin names from the target node. The BPIR compiler already knows the node's pin list at this point.

Example error:
```
Could not find target pin 'WidgetType' on node 'Create Widget'. Did you mean 'Class'? Available pins: OwningPlayer, Class, Outer, Style, [...]
```

Broader ask: cover every K2Node with this "did-you-mean" behaviour, since pin naming isn't consistent across the K2Node family (`WidgetClass` on AsyncAction, `Class` on CreateWidget, `ActorClass` on SpawnActor, `Struct` on BreakStruct, `Target` everywhere).

## History
- `#1-no-hint-on-wrong-pin-name` `OPEN` reporter — Hit during `W_MyReplaySelect.RebuildList` authoring. Used `WidgetType:` first (natural guess — the Create Widget node's tooltip says "Widget Type" in the editor UI). Error message did not hint the correct name. Had to `blueprint_decompile W_MyTrackSelect.CreateRemotelTrackItemWidget` to discover `Class:` is correct.
- `#2-did-you-mean-hint-added` `IN-REVIEW` developer — Enhanced the "Could not find target pin" error path in `BpirCompiler::Compile` wiring loop (Private/Compiler/BpirCompiler.cpp around line 4457). When an argument pin name fails to resolve on the target node, the compiler now collects all input (non-exec) pin names, picks a best-match candidate via case-insensitive substring containment (tie-broken by length delta), and emits "Did you mean 'X'? Available pins: ..." in the error. Applies to every node type that takes the generic-default wiring path, not just `K2Node_CreateWidget`. Broader "every K2Node + every error site" ask from the ticket was left out of scope (YAGNI).
- `#3-hint-fires-on-other-nodes` `IN-REVIEW` reporter — Additional confirmation: during replay-editor root-widget wiring, the hint also fired on `K2Node_CallFunction` for `UReplayEditorManualPlacementWidget::Begin(Target_1: ...)` → `Could not find target pin 'Target_1' on node 'Begin'. Did you mean 'Target'? Available pins: self, Target, InitialTransform`. Earlier in the same session the hint fired on `K2Node_CreateWidget(WidgetType:)` → `Available pins: Class, OwningPlayer`. The fix generalises beyond CreateWidget, as intended. Not marking DONE from the reporter side; leaving for the original tester.
- `#4-verified-hint-on-createwidget` `DONE` tester — Verified on a freshly-created `/Game/App/UI/Test/W_McpVerifyTemp`. Call 1: `compile_bpir` with `call K2Node_CreateWidget(WidgetType: ...)` → error `Could not find target pin 'WidgetType' on node 'Create Widget' Available pins: Class, OwningPlayer` (no Did-you-mean since "WidgetType" has no substring match against Class/OwningPlayer). Call 2: same with `WidgetClass:` → error `Could not find target pin 'WidgetClass' on node 'Create Widget'. Did you mean 'Class'? Available pins: Class, OwningPlayer`. The "Did you mean" hint + "Available pins" list both present as designed.
