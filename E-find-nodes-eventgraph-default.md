---
id: E-find-nodes-eventgraph-default
title: "`blueprint_graph_find_nodes` silently scopes to EventGraph by default"
status: DONE
severity: Medium
category: ergonomic
tags: []
---

# `blueprint_graph_find_nodes` silently scopes to EventGraph by default

When `graphName` is omitted, `blueprint_graph_find_nodes` only searches the EventGraph, not all graphs in the blueprint. For widgets with many function/macro graphs (10–15 is common for non-trivial UMG), this causes repeated empty-result false negatives when the target node lives in a function or macro body. The tool description does not clarify this default.

**Impact this session:**
During `W_MyReplaySelect` + `W_MyReplayListItem` cleanup, known-to-exist nodes (`Update Filter Buttons`, `Get Sender`, `Set Rename Visibility`, `Get DeleteButtonHovered`) returned empty from `blueprint_graph_find_nodes` without `graphName`, despite `blueprint_graph_list_graphs` reporting 12–15 graphs on each BP. Had to loop over graph names from `list_graphs` and re-query each one. Added substantial round-trip overhead.

**Workaround:**
After `blueprint_graph_list_graphs`, iterate and call `find_nodes` once per graph. Or pass an explicit `graphName` if you already know the target graph.

**Proposal:**
Either (a) change the default to search all graphs in the BP, returning a list tagged with `graphName` per hit, or (b) make the EventGraph-only default explicit in the tool description and surface it in `list_graphs` output as a hint ("Use find_nodes with graphName to scope to this graph"). Option (a) is preferable — it matches user intent when hunting orphans.

## History
- `#1-zero-hits-without-graphname` `OPEN` reporter — Hit during replay list BP cleanup. Without explicit `graphName`, searches for nodes buried in function/macro bodies returned zero hits even when the nodes clearly existed (compile errors confirmed them). Only after listing graphs and re-querying each by name could I locate the nodes for deletion.
- `#2-search-all-graphs-by-default` `IN-REVIEW` developer — When `graphName` is omitted, `blueprint.graph.find_nodes` now iterates all four graph collections on `UBlueprint` (UbergraphPages, FunctionGraphs, MacroGraphs, DelegateSignatureGraphs) using the same pattern as `list_graphs`. Each match entry is tagged with `graphName` and `graphKind`; top-level response includes `graphsSearched` and `scope`. `maxResults` is a global cap across graphs with a new default of 500 in the all-graphs path; `truncated: true` surfaces when the cap is hit. `ResolveBlueprintAndGraph` was left unchanged so other callers keep the EventGraph convenience.
- `#3-verified-all-graphs-searched` `DONE` tester — Verified: created `W_McpVerify_FindNodes`, added `MyHelperFunc` function with a `PrintString("UniqueMarker_FindNodes")` node placed via BPIR `entry function MyHelperFunc()`. `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"UniqueMarker_FindNodes"}` without `graphName` returned `matchCount:1`, match tagged with `graphName:"MyHelperFunc", graphKind:"function"`, top-level `scope:"all", graphsSearched:2`. Contrast call with `graphName:"EventGraph"` returned `matchCount:0, scope:"single", graphsSearched:1`. All-graphs default works; per-match tagging and top-level scope fields are present.
- `#4-component-bound-event-name-only` `DONE` reporter — Additional evidence (different finding, same tool): `mcp__editor_automation__.call path="blueprint.graph.find_nodes" args={"query":"SaveButton"}` on a graph containing a `K2Node_ComponentBoundEvent` titled "On Clicked (SaveButton)" returned zero matches. The widget variable name "SaveButton" appears in the node title's parens but does not appear in any of the indexed search fields for `K2Node_ComponentBoundEvent` (`nodeTitle` substring match doesn't surface it either). Same for searching "CommonButtonBaseClicked" — the underlying delegate signature class is invisible. Workaround: use `mcp__editor_automation__.call path="blueprint.graph.get_execution_flow" args={"includeAllEntryPoints":true,"entryPointsOnly":true}` and grep the entry-point list. Suggest `find_nodes` either expose the bound `componentName` (the `ComponentPropertyName` FName) as a queryable field, or add a top-level `componentName:` filter mirroring what `blueprint.graph_set_node_property` already accepts (per docs).
