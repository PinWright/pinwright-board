---
id: B-attach-decorator-properties-silently-ignored
title: "`behavior_tree.attach_decorator` reports success while silently ignoring the `properties` payload, leaving the decorator bound to the wrong Blackboard key"
status: OPEN
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
