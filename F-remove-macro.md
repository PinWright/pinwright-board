---
id: F-remove-macro
title: "`blueprint_remove_function` can't remove macros"
status: DONE
severity: High
category: feature
tags: []
---

# `blueprint_remove_function` can't remove macros

`blueprint.remove_function` accepts any graph name but only finds graphs of kind `function`/`event`. Passing the exact name of a graph of kind `macro` (as shown by `blueprint.graph.list_graphs`) returns `"Function not found — already removed or never existed"`. There is no separate `blueprint.remove_macro` tool.

**Impact this session:**
Replay list screen was cloned from `W_MyTrackSelect`, which has 6 BP macros (`LoadLocalTrackAssests`, `RebuildFilteredListOfLocalTracks`, `LoadRemoteTrackAssets`, `LoadUserCreatedTrackAssets`, `CheckAndContinueAfterLogin`, `UbdateTracksList`). None can be removed via MCP. Their bodies still contain call-function nodes to already-removed functions (`UpdateFilterButtons`, `AddTrackInfosToList`, `ClearList`, `UpdateListOfServerTracks`) and variable-get nodes for removed widgets (`W_TrackLoaderUtil`), causing compile errors that cannot be fully resolved via MCP.

**Workaround today:**
Delete individual nodes inside the macro via `mcp__editor_automation__.call path="blueprint.graph.get_nodes" args={"graphName":"UbdateTracksList"}` + `mcp__editor_automation__.call path="blueprint.graph.delete_node" args={...}` in a loop — leaves empty macro shells behind. Or have the user delete macros manually in the editor.

**Fix:**
Either (a) extend `blueprint.remove_function` to match macro graphs as well, or (b) add a new `blueprint.remove_macro` RPC that removes the macro graph from `UbergraphPages`/`FunctionGraphs`/`MacroGraphs` depending on internal storage.

## History
- `#1-macro-remove-not-found` `OPEN` reporter — Hit during replay list screen BP cleanup. `mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"UbdateTracksList"}` returned "not found" even though `mcp__editor_automation__.call path="blueprint.graph.list_graphs" args={...}` explicitly showed `{"name":"UbdateTracksList","kind":"macro"}`. Compile errors from orphan nodes inside the macro could not be eliminated via MCP alone.
- `#2-extended-to-macro-graphs` `IN-REVIEW` developer — Extended `FindBlueprintFunctionGraph` in `BlueprintFunctionHandler.cpp` to also walk `Blueprint->MacroGraphs`. Function graphs take precedence on name collision. Handler summary, param description, and idempotent-miss message updated to say "function or macro". Response JSON now includes `graphKind: "function"|"macro"`. `FBlueprintEditorUtils::RemoveGraph` handles macro-instance call-site cleanup, cosmetic cache flush, and redirector nulling natively — no additional cleanup logic needed.
- `#3-verified-macro-removal` `DONE` tester — Verified: created `W_McpVerify_RemoveMacro`, produced `TestMacro` via `mcp__editor_automation__.call path="blueprint.compile_bpir" args={...}` with `entry macro TestMacro() {}` (2 tunnel nodes). `mcp__editor_automation__.call path="blueprint.graph.list_graphs" args={...}` showed `{name:"TestMacro", kind:"macro", nodeCount:2}`. `mcp__editor_automation__.call path="blueprint.remove_function" args={"functionName":"TestMacro"}` returned `{success:true, graphKind:"macro", saved:true}` (no "Function not found"). Re-listing graphs showed only EventGraph remaining. Macro removal via `blueprint.remove_function` works end-to-end.
