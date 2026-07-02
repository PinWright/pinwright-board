---
id: B-node-id-dashed-guid-rejected
title: "Decompile warnings print nodeId as dashed GUID but graph.* resolver only matches undashed — pasting a reported nodeId yields NODE_NOT_FOUND"
status: IN-REVIEW
severity: Medium
category: bug
tags: [decompiler, graph-crud, guid-format, nodeid]
---

# Decompile warnings print nodeId as dashed GUID but graph.* resolver only matches undashed — pasting a reported nodeId yields NODE_NOT_FOUND

`blueprint.decompile` orphan warnings print node identity as a dashed GUID
(`nodeId=C6CCF838-462C-3BD1-97A7-74983DF743F3`) — `BpirDecompiler.cpp:904`
formats with `EGuidFormats::DigitsWithHyphens`. The shared node resolver
`FindNodeByIdOrName` (`BlueprintGraphHelpers.cpp:138-151`) matches with
`Node->NodeGuid.ToString()`, which defaults to `EGuidFormats::Digits`
(undashed), so the dashed string a warning hands you never matches:
`blueprint.graph.delete_node` returns `[NODE_NOT_FOUND]`; stripping the dashes
succeeds.

The decompiler warning is the *only* producer that emits the dashed form —
every other nodeId surface (create_node / find_nodes / get_node_details
results, `BuildPinJson` linkedTo labels) uses the undashed `Digits` default,
which is also what the resolver expects. Because `FindNodeByIdOrName` is shared
by all `graph.*` handlers (delete_node, get_node_details, set_node_property,
set_pin_default, connect_nodes, replace_node), the mismatch hits every
GUID-addressed graph edit fed a warning-sourced id, not just delete_node.

Introduced by `B-bpir-orphan-warning-diagnostics-lossy` (DONE), which added
node identity to the warning using `DigitsWithHyphens`.

**Workaround:** Strip the dashes from the reported nodeId before passing it
(`C6CCF838462C3BD197A774983DF743F3`).
**Fix:** Normalize in the shared resolver `FindNodeByIdOrName` — `FGuid::Parse`
the incoming id (auto-detects dashed vs. undashed) and compare `FGuid==FGuid`,
falling back to the existing name match; one edit covers all graph.* handlers.
Optionally also flip the warning at `BpirDecompiler.cpp:904` to the `Digits`
default so the whole ecosystem stays single-format.

## History
- `#1-producer-consumer-guid-format-mismatch` `OPEN` reporter — Producer/consumer disagree on GUID textual format. `blueprint.graph.delete_node {assetPath: "/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone", graphName: "EventGraph", nodeId: "C6CCF838-462C-3BD1-97A7-74983DF743F3"}` (dashed id copied verbatim from a `blueprint.decompile` orphan warning) → `[NODE_NOT_FOUND] Node not found.`; immediate retry with `"C6CCF838462C3BD197A774983DF743F3"` → success. Verified in source: warning emits `EGuidFormats::DigitsWithHyphens` at `BpirDecompiler.cpp:904`; shared resolver `FindNodeByIdOrName` string-compares `NodeGuid.ToString()` (default `Digits`) at `BlueprintGraphHelpers.cpp:138-151`, so the dashed form fails. Resolver is shared across all `graph.*` node-addressing handlers, so one normalize fixes all.
- `#2-implement-dashed-tolerant-resolver` `IN-REVIEW` developer — Verified in HEAD: producer emits dashed at `BpirDecompiler.cpp:979` (orphan) and `:1473` (cast<Unknown>); consumer `FindNodeByIdOrName` compared undashed `NodeGuid.ToString()` at `BlueprintGraphHelpers.cpp:144`. Fix: added pure shared matcher `BlueprintGraphHelpers::NodeGuidMatchesId(NodeGuid, Id)` (fast undashed compare, then `FGuid::Parse` + `FGuid==FGuid` — the same dashed/undashed-tolerant idiom already in `BehaviorTreeHandler.cpp:55-57` and `StateTreeAuthoringHandler.cpp:229`) and routed all three undashed-only node-GUID comparators through it: the shared resolver `FindNodeByIdOrName` (BlueprintGraphHelpers.cpp — covers delete_node/get_node_details/set_node_property/set_pin_default/connect_nodes/replace_node), plus the two inline comparators the ticket's "one edit" claim overlooked — the replace_node all-graphs fallback scan (`BlueprintGraphCrudHandler.cpp:812`) and the remove_event nodeId disambiguator (`BlueprintEventHandler.cpp:459`). Producer left dashed on purpose (parent `B-bpir-orphan-warning-diagnostics-lossy`'s orphan-distinguishability goal preserved) — the fix makes the consumer tolerant rather than reverting the parent. Regression test `PinWright.blueprint.graph.NodeIdDashedGuidResolves` (`Plugins/PinWright/Source/PinWright/Private/Tests/Blueprint/TestBlueprintGraphNodeIdGuidFormat.cpp`) builds an in-code Actor blueprint + node and asserts the dashed nodeId resolves (fails on revert), the undashed + name forms still resolve, and unrelated ids return null.
