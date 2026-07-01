---
id: E-widget-remove-widget-cascade-bound-events
title: "`widget_remove_widget` should cascade-delete ComponentBoundEvent nodes targeting the removed widget"
status: DONE
severity: Medium
category: ergonomic
tags: [widget-remove-widget, bound-event, cascade, cleanup]
---

# `widget_remove_widget` should cascade-delete ComponentBoundEvent nodes targeting the removed widget

When `widget_remove_widget` deletes a widget variable, any `K2Node_ComponentBoundEvent` nodes in the BP graphs that were bound to that widget's delegates are left behind as dangling orphans. The BP compiles with warnings like:

```
On Clicked (W_Create)  does not have a valid matching component!
On Clicked (BT_PupilsTracks)  does not have a valid matching component!
On Clicked (BT_MyTracks)  does not have a valid matching component!
```

And with errors when the bound event's downstream body references properties that were also removed. The caller must manually use `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={...}` for the stale events + `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` on each of them after the widget removal.

## Repro (observed this session)

Rebuilt `W_MyReplaySelect` by `asset_duplicate`-ing `W_MyTrackSelect` and stripping track-only widgets:

1. `mcp__editor_automation__.call path="widget.remove_widget" args={...}` for W_MyReplaySelect.FilterBox — returns `success: true`, widget tree no longer contains `FilterBox` or its children `BT_PupilsTracks`, `BT_MyTracks`.
2. `mcp__editor_automation__.call path="blueprint.compile" args={...}` — fails with 2 "does not have a valid matching component" warnings and 2 "Could not find a variable named BT_*" errors.
3. `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_ComponentBoundEvent"}` — reveals 4 stale bound events: `On Clicked (BT_PupilsTracks)`, `On Clicked (BT_MyTracks)`, `On Clicked (W_Create)`, `On Login Status Changed (W_LoginUtils)`.
4. Had to run `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` × 4 with individual node IDs to clean them up.

Same problem applies when stripping an entire track-era widget tree via `mcp__editor_automation__.call path="widget.import_xml" args={"mode":"replace",...}` — all the tree's ComponentBoundEvents survive because the event graph is a separate graph.

**Proposed fix:** `widget.remove_widget` should scan all graphs of the owning BP for `K2Node_ComponentBoundEvent` nodes whose `ComponentPropertyName` matches the removed widget's variable name, and delete them (along with their downstream exec chains, or at least the entry node so they become orphans detectable by `find_orphaned_nodes`). Response payload should report `cascadedBoundEventsRemoved: N`.

Also worth considering for `mcp__editor_automation__.call path="widget.import_xml" args={"mode":"replace",...}` — when a named widget is no longer in the new tree, any bound events referencing it should be cascaded.

## History
- `#1-stale-events-after-remove` `OPEN` reporter — Observed during replay subtask #4 rebuild. After `mcp__editor_automation__.call path="widget.remove_widget" args={...}` of `FilterBox`, `W_Create`, `SizeBox_1`, `SB_TableHeadersReview`, `W_TrackLoaderUtil`, `W_MyTrackListItem`, the EventGraph retained 4 stale ComponentBoundEvent nodes that required manual `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` cleanup. The BP would not compile until all 4 were removed.
- `#2-added-cascade-purge` `IN-REVIEW` developer — Extended the `widget.remove_widget` handler in `Private/Handlers/UI/WidgetHierarchyHandler.cpp` to cascade-purge stale bound events. Before `WidgetTree->RemoveWidget`, the handler recursively collects the target widget's variable name plus every descendant's variable name into a `TSet<FName>` (since `RemoveWidget` cascades to PanelWidget children). After removal it scans both `UbergraphPages` and `FunctionGraphs` for `UK2Node_ComponentBoundEvent` nodes whose `ComponentPropertyName` matches any removed name, walks their exec-reachable subgraph, and deletes the entry nodes + downstream chain via `FBlueprintEditorUtils::RemoveNode`. Response payload now includes `cascadedBoundEventsRemoved: N` and `removedWidgets: [names...]`.
- `#3-verified-cascade-delete` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`. Added `HudCheck2` CheckBox, bound a `widget_event HudCheck2.OnCheckStateChanged` body via `mcp__editor_automation__.call path="blueprint.compile_bpir" args={...}` (confirmed one `K2Node_ComponentBoundEvent_3` for HudCheck2 existed). `mcp__editor_automation__.call path="widget.remove_widget" args={"widgetPath":".../W_McpVerifyTemp","slotName":"HudCheck2"}` → `{success: true, removedWidget: "HudCheck2", cascadedBoundEventsRemoved: 1, cascadedCreateDelegatesRemoved: 0, removedWidgets: ["HudCheck2"]}`. Post-removal scan via `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_ComponentBoundEvent"}` returned zero matches. Cascade deleted the stale bound event + its downstream chain as advertised.
