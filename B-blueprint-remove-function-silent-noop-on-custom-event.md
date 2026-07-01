---
id: B-blueprint-remove-function-silent-noop-on-custom-event
title: "`blueprint_remove_function` silent no-op when name targets a custom event"
status: DONE
severity: Medium
category: bug
tags: [remove-function, custom-event, silent-noop, name-routing, idempotent-mismatch]
---

# `blueprint_remove_function` silent no-op when name targets a custom event

`mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"SomeName"}` returns `success: true` with `note: "Function or macro not found — already removed or never existed"` when `SomeName` is actually a `K2Node_CustomEvent` in the EventGraph (not a function or macro graph). The custom event remains in the BP. The caller has to know the entry kind ahead of time and switch to `blueprint.remove_event` instead.

## Repro (observed this session, `W_RenameReplay`)

Cloned `W_RenameTrack` → `W_RenameReplay`. The clone has both:
- A function `UpdateTrackInfo` (in `Functions` graphs)
- A custom event `ApplyTrackInfo` (in `EventGraph`)

Both surface in `blueprint.inspect` — `UpdateTrackInfo` under `functions`, `ApplyTrackInfo` under `events` with `eventType: "custom"`. From the user's perspective they're both addressable by name.

```
mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"UpdateTrackInfo"}  →  removes successfully (success:true, graphKind:"function")
mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"ApplyTrackInfo"}   →  success:true, note:"Function or macro not found — already removed or never existed"
```

The second call leaves `ApplyTrackInfo` in the EventGraph as a live `K2Node_CustomEvent` with a `Set Text` exec downstream. Subsequent `mcp__editor_automation__.call path="blueprint.graph.get_execution_flow" args={"includeAllEntryPoints":true}` confirms it's still listed.

## Impact

A caller migrating an asset from one schema to another (e.g., this session: replacing TrackInfo-flavored events with EditorReplay-flavored ones) reasonably calls `remove_function` for every name they want gone, gets four `success:true` responses, and is left with stale custom events that fail compilation downstream when the variables they reference get deleted. Diagnosing requires re-running `blueprint.inspect` and noticing the entry is in `events` instead of `functions` — easy to miss when the name was in both lists at start.

The `idempotent` framing of the `note` is also misleading: from the caller's perspective, the entry **does** exist, just under a different conceptual namespace.

**Workaround:** check `blueprint.inspect` first; if name appears under `events`, use `blueprint.remove_event` with the same name. Or always call both removers and accept the noisier no-op.

**Proposal:** when `remove_function` can't find the named function/macro, scan custom events too and either:
- (preferred) remove the matching `K2Node_CustomEvent` + its exec subgraph (same pattern `blueprint_remove_event` already implements per `B-remove-event-inconsistent` #5), and return `removedNodeCount > 0` with a `kind: "custom_event"` field so the caller knows what happened.
- OR keep the no-op but change the `note` to "No function or macro by that name. Found a custom event with the same name — use blueprint.remove_event instead." — explicit hint, no silent state divergence.

## History
- `#1-initial-repro` `OPEN` reporter — Session repro on `/App/App/UI/LobbyAndMenu/Popups/W_RenameReplay` (cloned from W_RenameTrack). `mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"UpdateTrackInfo"}` removed the function correctly; `mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"ApplyTrackInfo"}` returned the "not found" note while the custom event was still live and downstream `Set Text` node still wired. Worked around by `mcp__editor_automation__.call path="blueprint.graph.get_execution_flow" args={...}` to find the event's nodeId, then `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` for both the event entry and its exec body.
- `#2-cascade-remove-custom-event` `IN-REVIEW` developer — Extended `blueprint.remove_function` in `BlueprintFunctionHandler.cpp` to scan the event graph for a matching `UK2Node_CustomEvent` when the function/macro lookup fails, then cascade-remove via the existing `BlueprintHandlerUtils::GetEffectiveFunctionNameForRemoval` + `CascadeRemoveStaleCreateDelegates` utilities (mirrors `blueprint.remove_event`'s pattern). Response now carries `kind: "custom_event"` / `graphKind: "event"` / `removedNodeCount` so callers can distinguish the branch. Regression test `FBlueprintRemoveFunctionCascadesCustomEventTest` in `TestBlueprintRemoveFunctionCascadesCustomEvent.cpp` asserts the custom event is gone after the call and the response carries the new markers.
- `#3-verified-cascade-custom-event` `DONE` tester — Verified on `/Game/App/UI/Test/W_McpVerifyTemp`: emitted `entry custom_event T_remove_target() { call PrintString(InString: "hi") }` via `mcp__editor_automation__.call path="blueprint.compile_bpir" args={...}`, then called `mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"T_remove_target"}`. Response: `success: true, kind: "custom_event", graphKind: "event", removedNodeCount: 2, cascadedCreateDelegatesRemoved: 0, saved: true`. Pre-fix response would have been `success: true, note: "Function or macro not found — already removed or never existed"` with the custom event still live. Cascade-remove path engaged correctly.
