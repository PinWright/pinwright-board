---
id: B-attach-decorator-properties-silently-ignored
title: "`behavior_tree.attach_decorator` reports success while silently ignoring the `properties` payload, leaving the decorator bound to the wrong Blackboard key"
status: IN-REVIEW
severity: High
category: bug
tags: [behavior-tree, decorator, silent-success]
---

# `behavior_tree.attach_decorator` reports success while silently ignoring the `properties` payload, leaving the decorator bound to the wrong Blackboard key

`behavior_tree.attach_decorator` accepts a `properties` object documented as "Key-value properties to
set on the decorator instance". For `BTDecorator_Blackboard` it creates and attaches the decorator,
returns a `nodeId` and no error — and applies **none** of the blackboard-condition properties.

## Reproduction (this checkout, 2026-09-03)

    behavior_tree.attach_decorator {
      assetPath: "/Game/FPS/AI/BT_Enemy",
      parentNodeId: "BehaviorTreeGraphNode_Composite_4",
      decoratorClass: "BTDecorator_Blackboard",
      properties: { BlackboardKey: "bDwellStalled", NotifyObserver: "OnValueChange",
                    FlowAbortMode: "Self", KeyQuery: "IsUnset" }
    }
    -> { nodeId: "BA1591414071FBB2EDCA749340CCAACB", parentNodeId: "871E64DD...", ... }   # no error

`behavior_tree.decompile` immediately after:

    decorator Blackboard `Blackboard Based Condition` (BlackboardKey: SelfActor, FlowAbortMode: "Self")

`FlowAbortMode` was applied. `BlackboardKey` was **not** — it silently defaulted to the blackboard's
first key (`SelfActor`), and `NotifyObserver` / `KeyQuery` are absent entirely.

## Why this is the dangerous shape

The decorator is attached to the right parent, the call returns success, and the tree still compiles
and runs. The branch is simply gated on the wrong key — here `SelfActor`, which is always set, so the
condition is permanently true and the intended abort never fires. Nothing in the response, the
compile, or a casual decompile read flags it; you only catch it by decompiling and *reading the key
name*, or by watching the behaviour fail in PIE. That is a silent success-with-no-effect on the exact
property the caller most needs.

## Workaround

Set the fields afterwards on the runtime decorator object in Python — note `blackboard_key` is a
struct that must be read, mutated and written back, and `selected_key_id` must match the key's index
in the Blackboard's `keys` array or the decorator resolves nothing at runtime:

    d = unreal.find_object(bt, 'BTDecorator_Blackboard_7')
    d.modify(True)
    key = d.get_editor_property('blackboard_key')
    key.set_editor_property('selected_key_name', 'bDwellStalled')
    key.set_editor_property('selected_key_id', <index in BB.keys>)
    d.set_editor_property('blackboard_key', key)
    d.set_editor_property('notify_observer', unreal.BTBlackboardRestart.VALUE_CHANGE)
    d.set_editor_property('basic_operation', unreal.BasicKeyOperation.NOT_SET)

## Ask

Apply the `properties` payload (at minimum `BlackboardKey`, `NotifyObserver`, `KeyQuery` /
`BasicOperation`, and the arithmetic variants), resolving `selected_key_id` from the owning
Blackboard; and return an explicit error listing any key in `properties` the handler did not apply,
rather than reporting a bare success.

## Fix

Confirmed against source. `properties` *was* applied — through the shared writer — but the attach
path passed **no** drop list (`BehaviorTreeHandler.cpp:529`, comment: "pass no out-list to discard
them"), so every rejected key vanished. Two keys reject for `BTDecorator_Blackboard`:
`BlackboardKey` (a bare string reaches `ApplyJsonValueToProperty`'s struct branch and fails both the
JSON parse and the `ImportText` fallback) and `KeyQuery` (a details-panel DisplayName, not a
UPROPERTY — the property is `BasicOperation`). A third half-write existed even for keys that *did*
land: `UBTDecorator_Blackboard` maps `BasicOperation`/`ArithmeticOperation`/`TextOperation` onto the
`OperationType` byte its evaluation reads only inside `PostEditChangeProperty`, which a raw
reflection store never fires.

Design: everything still goes through the one shared writer (`ApplyJsonValueToProperty`, the same
one `property.set` uses). Added around it — never instead of it — (a) a bare string on an
`FBlackboardKeySelector` is normalized to `{"SelectedKeyName": ...}` so the existing converter does
the write, (b) `ResolveSelectedKey` against the tree's `BlackboardAsset` (the BT editor's own step,
`UBehaviorTreeGraph::UpdateBlackboardChange`), with a name the blackboard lacks reported as a failed
key listing the available names and the selector restored, (c) `PinWright::NotifyPropertyChanged`
per applied key in the leaf/member shape the engine branches match on, (d) key selectors applied
before other keys, since the engine resets `IntValue`/`StringValue` on an enum-key change.

Files changed:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/AI/BehaviorTreeHandler.cpp` —
  `ApplyBTNodeProperties` now returns a per-key `FBTNodePropertyReport` (applied + failures) and
  takes the owning `UBehaviorTree`; new `AsBlackboardKeySelectorProperty`,
  `CollectBlackboardKeyNames`, `ResolveBlackboardKeySelector`, `ReadBTPropertiesField` (a non-object
  `properties` is now an error, not a silent drop) and `SendBTPropertyFailureError` (the shared
  `droppedFields` rejection). `attach_decorator` / `attach_service` hard-fail with
  `INVALID_PROPERTY` on any dropped key and **un-attach the subnode** (`rolledBack: true`); success
  carries `appliedProperties`. `set_node_properties` keeps its existing error contract and gains
  `appliedProperties`.
- `Plugins/PinWright/Source/PinWright/Private/Tests/Gameplay/TestBehaviorTreeAttachSubnodes.cpp` —
  the two tests below plus a Blackboard+BT fixture helper.
- `Plugins/PinWright/Docs/wiki-src/behavior_tree.md` — the `properties` contract under "Decorators
  and services" (bold label, not `###`, so the sections below it still render).

Reviewer verification:

1. `PinWright.behavior_tree.attach_decorator.AppliesBlackboardKeyProperty` — attaches a Blackboard
   decorator with `{BlackboardKey: "bDwellStalled", BasicOperation: "NotSet", NotifyObserver:
   "ValueChange"}` and asserts `SelectedKeyName` is the requested key, `SelectedKeyID` equals the
   blackboard's id for it (the resolve), and `OperationType == EBasicKeyOperation::NotSet` (the
   change notification ran *and* selectors were applied first).
2. `PinWright.behavior_tree.attach_decorator.RejectsUnknownPropertyName` — attaches with the
   reported `KeyQuery` key and asserts `INVALID_PROPERTY`, `droppedFields` naming it,
   `rolledBack: true`, and that the parent node ends with zero decorators.
3. By hand: repeat the reproduction above, then `behavior_tree.decompile` — the decorator line must
   read `BlackboardKey: bDwellStalled`. Re-run with `KeyQuery` in the payload and confirm the call
   errors and no decorator is left on the parent.

Not compiled and not run — a separate compile/test pass follows.
