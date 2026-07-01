---
id: E-remove-event-multi-match-dedup
title: "`blueprint_remove_event` can't distinguish multiple events with same delegate signature name"
status: DONE
severity: Low
category: ergonomic
tags: [blueprint-remove-event, dedup, component-bound-event]
---

# `blueprint_remove_event` can't distinguish multiple events with same delegate signature name

`blueprint_remove_event` takes `eventName` as a string. When a BP has multiple `K2Node_ComponentBoundEvent` nodes that share the same underlying delegate signature name (common when several buttons in the same widget all emit `CommonButtonBaseClicked__DelegateSignature`), there's no way to target a specific one — the tool either removes the first match or is ambiguous.

In the session's `W_MyReplaySelect` cleanup, the decompile showed three indistinguishable entries:

```
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) { set FilterMyTracks = false }
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) { set FilterMyTracks = true }
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) { ...push W_LocationsSelectWindow... }
```

All three needed deletion but `mcp__editor_automation__.call path="blueprint.remove_event" args={"eventName":"CommonButtonBaseClicked__DelegateSignature"}` can't target one without removing them all (or the first one).

**Workaround (session-tested):** Fall back to `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_ComponentBoundEvent"}` to discover the node IDs, then `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` on each specific ID. Works, but it's 2 extra tool calls per event + the caller has to visually disambiguate by node title (`On Clicked (BT_PupilsTracks)` vs `On Clicked (BT_MyTracks)`).

**Proposed fix:** Accept an optional `componentName` parameter on `blueprint_remove_event` — e.g. `componentName: "BT_PupilsTracks"` disambiguates which `K2Node_ComponentBoundEvent` to remove. Alternative: accept a node title pattern (`"On Clicked (BT_PupilsTracks)"`) or a nodeId.

## History
- `#1-same-delegate-ambiguity` `OPEN` reporter — Observed this session during `W_MyReplaySelect` cleanup. Three same-named `CommonButtonBaseClicked__DelegateSignature` entries (all bound to different buttons) could not be individually targeted via `mcp__editor_automation__.call path="blueprint.remove_event" args={...}`. Worked around by enumerating via `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={...}` and deleting by node ID.
- `#2-added-component-name-param` `IN-REVIEW` developer — Added optional `componentName` and `nodeId` params to `blueprint.remove_event` in BlueprintEventHandler.cpp. When provided, candidate EventRootNodes are filtered by the owning component's property name (ComponentBoundEvent) or by NodeGuid. Default behavior unchanged when neither is set. Pinned by FBlueprintRemoveEventComponentNameFilterTest in TestBlueprintRemoveEventDedup.cpp.
- `#3-verified-disambiguators` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`. Added two CheckBoxes (`HudCheck`, `HudCheck2`) and wired a `widget_event OnCheckStateChanged` body on each via `mcp__editor_automation__.call path="blueprint.compile_bpir" args={...}`, producing two `K2Node_ComponentBoundEvent` nodes with the same delegate signature but different `ComponentPropertyName`. (1) `mcp__editor_automation__.call path="blueprint.remove_event" args={"eventName":"OnCheckStateChanged","componentName":"HudCheck2"}` → `removedNodeCount: 2, cascadedCreateDelegatesRemoved: 0`; `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_ComponentBoundEvent"}` after → only the HudCheck entry remains. (2) `mcp__editor_automation__.call path="blueprint.remove_event" args={"eventName":"OnCheckStateChanged","nodeId":"<HudCheck's ComponentBoundEvent GUID>"}` → `removedNodeCount: 2`; subsequent scan shows both bound events gone. Both disambiguators work; schema exposes them as optional params.
