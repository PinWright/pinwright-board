---
id: B-widget-event-stale-class-ref-upsert-fail
title: "`widget_event` append does not upsert when referenced WidgetClass was deleted+recreated"
status: DONE
severity: High
category: bug
tags: [bpir, widget-event, upsert, class-resolution, stale-ref]
---

# `widget_event` append does not upsert when referenced WidgetClass was deleted+recreated

When a BPIR `entry widget_event <Widget>.<Event>()` body references an external `WidgetClass` (via `K2Node_AsyncAction(WidgetClass: /App/.../W_Target_C, ...)`) and that `W_Target` asset was since `asset_delete`'d and `asset_duplicate`'d (same path, same name, new generated-class GUID), re-compiling the same BPIR with `mode: "append"` does **not** upsert the existing `K2Node_ComponentBoundEvent`. Instead it creates a second bound event on the same source widget, leaving both wired to the button's click delegate — every click fires both (old and new).

The old bound event still has broken internal references to the now-deleted class GUID, so its downstream nodes (old `K2Node_AsyncAction`, old `K2Node_DynamicCast` pointing at the stale class) fail to compile with errors like:

```
COMPILER ERROR: failed building connection with 'This cast has an invalid target type (was the class deleted without a redirect?).' at PushContentToLayerForPlayer
Event Dispatcher has no property  Unbind all Events from OnOpenReplayRequested
Event Dispatcher has no property  Bind Event to OnOpenReplayRequested
```

BPIR reports `success: true` for the new compile (new chain was emitted fine), but `blueprint_compile` fails because of the co-resident broken chain.

## Repro (observed this session)

1. On `W_LyraFrontEnd` compile `entry widget_event W_ReplayEditorButton.OnClicked() { ...cast<W_MyReplaySelect_C>(...); bind_dispatcher OnOpenReplayRequested(...) }` — creates `K2Node_ComponentBoundEvent_7` with the bound body. Compile clean.
2. `asset_delete /App/App/UI/LobbyAndMenu/W_MyReplaySelect`
3. `asset_duplicate /App/App/UI/LobbyAndMenu/W_MyTrackSelect → /App/App/UI/LobbyAndMenu/W_MyReplaySelect` (same path, different generated-class GUID now).
4. Re-compile the same BPIR `entry widget_event W_ReplayEditorButton.OnClicked() { ... }` on `W_LyraFrontEnd` — `success: true`, 11 new nodes created. But `blueprint_graph_find_nodes` now shows **two** `On Clicked (W_ReplayEditorButton)` bound events: `K2Node_ComponentBoundEvent_7` (old, with broken cast + broken bind_dispatcher on dead class GUID) and `K2Node_ComponentBoundEvent_8` (new, correctly wired).
5. `blueprint_compile` errors on the old chain.

**Workaround (session-tested):** After the BPIR upsert, manually `blueprint_graph_find_nodes` for duplicate bound events on the same widget, then `blueprint_graph_delete_node` on the older ID. Also run `blueprint_graph_find_orphaned_nodes` and delete the orphan `GetOwningPlayer`, `K2Node_Self`, `K2Node_DynamicCast`, `K2Node_CreateDelegate` nodes that were downstream of the old broken event.

**Proposed fix:** `widget_event` upsert should match on `{widget_variable_name, event_name}` and delete the existing `K2Node_ComponentBoundEvent` + its downstream exec chain before emitting the new body, regardless of whether the old node's internal class references are still resolvable. Currently the matcher may be keying on the binding's class GUID, which fails after delete+recreate.

Related but distinct from:
- `B-compile-bpir-retry-duplicates` (DONE) — that's RPC retry mid-write; this is successful replies-but-stale-class.
- `B-widget-event-xml-imported-button` (DONE) — that's new-widget XML import; this is existing widget with stale class target.

## History
- `#1-duplicate-bound-event-stale-class` `OPEN` reporter — Observed this session during replay subtask #4 rebuild. `W_LyraFrontEnd.W_ReplayEditorButton.OnClicked` bound event upsert failed after `W_MyReplaySelect` was deleted + duplicated from `W_MyTrackSelect`. Two `K2Node_ComponentBoundEvent` nodes ended up on the same button, both referencing the click delegate, the old one carrying broken cast + bind_dispatcher references to the dead class GUID. Manual `blueprint_graph_delete_node` cleanup required.
- `#2-phase0-pre-upsert-added` `IN-REVIEW` developer — Added a new unconditional Phase 0-pre pass in `BpirCompiler::Compile` (Private/Compiler/BpirCompiler.cpp) that runs in all modes (Default, Replace, Extend). For every `ComponentEvent`/`WidgetEvent` block in the incoming BPIR, it scans `UbergraphPages` for existing `UK2Node_ComponentBoundEvent` nodes whose `ComponentPropertyName` matches the block's component and whose `DelegatePropertyName` matches the block's event name (case-insensitive, space-stripped parity with the creation-side matcher in `FCodeNodeEmitter::CreateComponentEventNode`). Matching entry nodes and their exec-reachable subgraphs are collected via `CollectSubgraphNodes` and deleted via `FBlueprintEditorUtils::RemoveNode`. Root cause: the existing Replace-mode scan was keyed on `UK2Node_CustomEvent` casts and never matched `UK2Node_ComponentBoundEvent` (which derives from `UK2Node_Event`, not `CustomEvent`). Additionally, Phase 0 was gated on `Replace` mode, so Default/Append never deleted old bound events, which is unsafe — `(ComponentPropertyName, DelegatePropertyName)` is inherently unique per delegate and re-emitting always produces a duplicate.
- `#3-verified-phase0-upsert-unique` `DONE` tester — Verified Phase 0-pre upsert uniqueness on `/Game/App/UI/Test/W_McpVerifyTemp`. With no pre-existing HudCheck bound event, ran `mcp__editor_automation__.call path="blueprint.compile_bpir" args={"mode":"append",...}` twice on the same `entry widget_event HudCheck.OnCheckStateChanged` with different body contents. After the second compile, `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"K2Node_ComponentBoundEvent","graphName":"EventGraph"}` returned `matchCount: 1` — only the second emission survived; the first one was swept by Phase 0-pre as designed. Without this fix, append-mode would have produced two bound events on the same `(ComponentPropertyName, DelegatePropertyName)` tuple, which is the root pathology behind the original delete+duplicate stale-class failure. Full delete+duplicate+recompile chain not live-tested (would require asset churn on an external BP; W_LyraFrontEnd is currently blocked by the re-opened B-bpir-class-resolve-reentrant-crash), but the key Phase 0-pre behavior the fix adds is confirmed.
