---
id: B-node-id-dashed-guid-rejected
title: "Decompile warnings print nodeId as dashed GUID but graph.* resolver only matches undashed — pasting a reported nodeId yields NODE_NOT_FOUND"
status: IN-REVIEW
severity: Medium
category: bug
tags: [decompiler, graph-crud, guid-format, nodeid]
encounters: 2
lastSeen: 2026-09-05T00:00:00Z
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
- `#3-cross-surface-format-mismatch-blocks-correlation` `IN-REVIEW` WEAPONS-critic — Additional evidence, status unchanged; **the consumer half may well be fixed, but the producer split this ticket's own "optionally" clause names is still live and it blocks a second use case the ticket does not yet state.** Measured in a WEAPONS critic review round 3 on the `/Game/FPS/Weapons/` Blueprints: `blueprint.graph.find_orphaned_nodes` prints node ids **unhyphenated** (`A7E232894DB899451DC2E98A282FB7A8`) while `asset.dump`'s `bpir.txt` prints them **hyphenated** (`67702A60-4CAA-F084-6B67-5CBB446B2155`). `#2`'s fix makes both forms *resolve* when handed back to a `graph.*` verb, which closes the paste-a-warning-id path in the title. It does not make the two surfaces **cross-referenceable**: correlating a BPIR warning against the finder's node list is a string comparison between two response bodies, no resolver involved, and it fails on every row. Any consumer diffing "what BPIR complains about" against "what the finder reports" — which is exactly the check `B-orphan-finder-vs-decompiler-disagree` exists to make possible, and which that ticket's own `#10` records the plugin's *own test* getting wrong the same way ("the new test searched BPIR's hyphenated warning GUID with the finder's compact JSON GUID form, so its locally reconstructed BPIR set was empty despite the warning being present") — must normalise by hand first. When the project's own regression test has already been bitten by this, the format split is not a cosmetic preference. **One observation deliberately not claimed as evidence:** the two surfaces also reported different GUID *values* for a node with the same title in the same graph, but the graph provably changed between the calls (421 → 420 nodes), so that difference has an ordinary explanation and is recorded, not asserted. The **formatting** difference is not timing-dependent and reproduces on every call. Ask, added to this ticket rather than filed separately since it is the same producer/consumer split: take the "optionally" clause in the Fix section and make it the fix — emit the `Digits` (unhyphenated) default from `BpirDecompiler`'s warning sites so one format exists ecosystem-wide, keeping `#2`'s tolerant resolver as the compatibility layer for ids already written into logs and tickets. Severity unchanged at Medium. No plugin source was opened for this entry; the `file:line` citations above are quoted from this ticket's and `B-orphan-finder-vs-decompiler-disagree`'s existing history, not re-derived.
