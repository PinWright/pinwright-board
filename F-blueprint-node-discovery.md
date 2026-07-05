---
id: F-blueprint-node-discovery
title: "blueprint.graph: action-database node discovery with pin-context filtering"
status: OPEN
severity: Medium
category: feature
tags: [blueprint, graph, discovery, action-database, parity-ue58]
---

# blueprint.graph: action-database node discovery with pin-context filtering

BPIR and `blueprint.graph.*` node creation require the agent to already know node/function names; resolution is a name-cascade (see bpir compiler internals section 6). There is no way to ask the editor "what nodes exist matching X" or, critically, "what can legally connect to this pin". Agents fall back to guessing K2 spellings and retrying compile errors.

UE 5.8 parity evidence: `find_node_types(graph, filter, context_pins)` / `find_node_categories` / `get_node_type_pins` (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\EditorToolset\Content\Python\editor_toolset\toolsets\blueprint.py:752`) over `UBlueprintGraphEditor::ListAvailableNodes` -> `FBlueprintActionDatabase` + `FBlueprintActionFilter` (`Engine\Source\Editor\BlueprintEditorLibrary\Private\BlueprintEditorLibrary\BlueprintGraphEditor.cpp:1788`). The context_pins parameter filters candidates by pin-type compatibility. The underlying engine APIs (`FBlueprintActionDatabase`, `FBlueprintActionFilter`) exist on UE 5.3+, so this ports across our support matrix; only Epic's convenience wrapper is 5.8-only.

Proposed scope:
- `blueprint.graph.find_node_types(blueprint, graph, filter, contextPins?)` - substring/category search over the action database scoped to the target graph, optional pin-context compatibility filtering; returns stable type ids usable by node-creation RPCs and suggested BPIR spellings where mappable.
- `blueprint.graph.get_node_type_pins(typeId)` - expected pin list for a candidate without creating it (spawn-and-scan into a transient graph is acceptable v1, as Epic does).

Acceptance: querying with a float output pin as context returns math nodes accepting float and excludes exec-only nodes; a returned type id round-trips into successful node creation.

## History
- `#1-no-node-discovery` `OPEN` reporter — No action-database search or pin-compatibility discovery; agents guess K2 names and retry. Epic 5.8 find_node_types with context_pins over FBlueprintActionDatabase (engine API available 5.3+) is the parity target.
