---
id: B-bpir-orphan-warning-diagnostics-lossy
title: "BPIR orphan warnings omit node identity despite find_orphaned_nodes having it"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, orphan-diagnostics]
---

# BPIR orphan warnings omit node identity despite find_orphaned_nodes having it

BPIR orphan warnings only include a display title. Repeated nodes with the same title are indistinguishable, and the dump does not include graph, node id, class, coordinates, pins, or why traversal failed, even though the live orphan finder can expose that information.

**Workaround:** Re-run `blueprint.graph.find_orphaned_nodes` and `blueprint.graph.get_node_details` on the asset.

**Fix:** Include graph name, node id, node type, title, and coordinates in the decompiler warning text, or emit structured diagnostics alongside `bpir.txt`.

## History
- `#1-fresh-app-game-dump-audit` `OPEN` reporter — Fresh 2026-05-09 App/Game dumps contain 1,157 `Orphaned node not reachable from any entry point` warning lines across 164 BPIR files. The BPIR warning only includes display title, so repeated nodes like `SetVisibility` are indistinguishable. Live dry-run `blueprint.graph.find_orphaned_nodes` on `/App/App/UI/W_AppMapEditor_ActionPanel` returns nodeId, nodeType, graphName, hasExecPins, and count; `get_node_details` adds x/y and pins. Decompiler warning output should include at least graphName, nodeId, nodeType, title, and coordinates, or emit structured diagnostics alongside `bpir.txt`.
- `#2-orphan-warning-enrichment` `IN-REVIEW` developer — Replaced the title-only printf at `BpirDecompiler.cpp` orphan-sweep emit site with an enriched format including graphName, node class, title, GUID (`DigitsWithHyphens`), and `@(x,y)` coordinates; leading sentence prefix preserved for existing prefix-matchers. Extended `TestBpirDecompilerOrphanWarnings.cpp` with `FBpirDecompilerOrphanWarningIncludesNodeIdentityTest` (`EditorAutomationRpcGateway.bpir.decompiler.OrphanWarningIncludesNodeIdentity`): adds two unwired same-titled PrintString orphans with distinct positions, asserts count==2, textual distinctness, and per-node GUID/class/title/coordinate substrings. Counterfactual: reverting to title-only collapses both warnings to the identical string `"Orphaned node not reachable from any entry point: Print String"`, causing the distinctness assertion and all GUID/coordinate substring assertions to fail.
- `#3-verify-fix` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/W_AppMapEditor_ActionPanel`. All 5 orphan warnings now contain graphName (`EventGraph`), node class (`K2Node_CallFunction`), title (e.g. `'Set Visibility'`), `nodeId=<GUID>`, and `@(x,y)` coordinates. Three repeated `Set Visibility` orphans are now individually distinguishable by GUID and coordinates (e.g. `nodeId=A7088E2D-... @(2960,5584)` vs `nodeId=8C0AD2A7-... @(3350,5584)` vs `nodeId=68E1093F-... @(2960,6048)`).
