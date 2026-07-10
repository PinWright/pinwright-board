---
id: F-blueprint-node-discovery
title: "blueprint.graph: action-database node discovery with pin-context filtering"
status: IN-REVIEW
severity: Medium
category: feature
tags: [blueprint, graph, discovery, action-database, parity-ue58]
claimedBy: fuzz2
claimedAt: 2026-07-10T12:23:00.3261702+03:00
---

# blueprint.graph: action-database node discovery with pin-context filtering

Building a Blueprint graph node-by-node (via `blueprint.graph.create_node` or
`blueprint.compile_bpir`) requires the agent to already know the node/function it
wants. Two PARTIAL discovery surfaces exist today, but neither answers "what can
legally connect to THIS pin":

- `blueprint.graph.list_node_types` (`BlueprintGraphInspectionHandler.cpp:1282`) —
  a flat `TObjectIterator<UClass>` over non-abstract `UK2Node` container classes,
  returning `{className, displayName}` only. No filter, no pin context, no
  per-spawner detail.
- `blueprint.build_api_index` -> `blueprint.search_api`
  (`BlueprintApiIndexHandler.cpp:27,234`) — keyword search over reflected
  `FUNC_BlueprintCallable` **UFunctions**. This resolves "intent -> Kismet function
  name" for the `CallFunction` case (and the sibling docs tickets
  `E-create-node-operator-symbol-discovery` / `E-bpir-pure-fn-name-undiscoverable`
  redirect agents here), but it indexes functions ONLY — not the rest of the action
  database (flow control, variable get/set, casts, events, macros) — and it has no
  pin-type-compatibility filter.

The genuinely-absent capability is **pin-context-aware discovery over the full
Blueprint action database**: given an output pin of a certain type, "what spawnable
nodes accept it?" — the query the editor's drag-off-a-pin context menu answers. No
RPC provides this; agents fall back to guessing K2 spellings and retrying compile
errors.

UE 5.8 parity evidence: `find_node_types(graph, filter, context_pins)` /
`get_node_type_pins`
(`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\EditorToolset\Content\Python\editor_toolset\toolsets\blueprint.py:752`)
over `UBlueprintGraphEditor::ListAvailableNodes` -> `FBlueprintActionDatabase` +
`FBlueprintActionFilter`
(`Engine\Source\Editor\BlueprintEditorLibrary\Private\BlueprintEditorLibrary\BlueprintGraphEditor.cpp:1788`).
The context_pins parameter filters candidates by pin-type compatibility. The
underlying engine APIs (`FBlueprintActionDatabase`, `FBlueprintActionFilter`,
`UBlueprintNodeSpawner`) exist on UE 5.3+, so this ports across our support matrix;
only Epic's convenience wrapper is 5.8-only.

Proposed scope:
- `blueprint.graph.find_node_types(assetPath, graphName?, filter?, contextPins?)` -
  substring search over the action database scoped to the target graph, with optional
  pin-type-compatibility filtering against `contextPins`; returns stable type ids
  usable by `get_node_type_pins`. This **complements, does NOT replace**
  `blueprint.search_api`: search_api stays the keyword->UFunction tool, while
  find_node_types adds action-database breadth (non-function nodes) + pin-context
  filtering that search_api/list_node_types cannot do.
- `blueprint.graph.get_node_type_pins(typeId)` - expected pin list for a candidate
  type without placing it in a user graph (spawn-and-scan a template node, as Epic
  does).

Acceptance: querying with a float output pin as context returns math nodes accepting
float and excludes exec-only nodes; a returned type id round-trips through
`get_node_type_pins` to a pin list.

## History
- `#1-no-node-discovery` `OPEN` reporter — No action-database search or pin-compatibility discovery; agents guess K2 names and retry. Epic 5.8 find_node_types with context_pins over FBlueprintActionDatabase (engine API available 5.3+) is the parity target.
- `#2-reword-and-go` `IN-REVIEW` developer — Reworded then GO (verified vs source; the adversarial wontfix rejected). The original body claimed "no way to ask what nodes exist matching X" — materially overstated: `blueprint.search_api`/`build_api_index` (keyword over `FUNC_BlueprintCallable` UFunctions only, `BlueprintApiIndexHandler.cpp:86`) and `list_node_types` (flat `UK2Node` class enum, `BlueprintGraphInspectionHandler.cpp:1282`) already give PARTIAL discovery. Neither provides action-database breadth (non-function nodes) or pin-type-compatibility filtering, and no `find_node_types`/`get_node_type_pins`/`FBlueprintActionDatabase` discovery RPC exists — so the pin-context delta is genuinely absent. Kept title/Medium/feature; rewrote the motivating prose to scope to that delta and cross-reference search_api/list_node_types. Implementing `find_node_types` + `get_node_type_pins` over `FBlueprintActionDatabase` with `contextPins` pin-compat filtering. Scope note: the optional "suggested BPIR spellings where mappable" sub-feature is intentionally dropped from v1 — BPIR intent->name discovery is already the sibling `E-bpir-pure-fn-name-undiscoverable` docs redirect's job; it should not ship as part of this capability.
