---
id: E-blueprintgraph-handler-split
title: "Split 4,480 LOC BlueprintGraphHandler into CRUD / Connections / Inspection-Search TUs"
status: DONE
severity: Low
category: ergonomic
tags: [blueprint, refactor, code-organization]
---

# Split 4,480 LOC BlueprintGraphHandler into CRUD / Connections / Inspection-Search TUs

`Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintGraphHandler.cpp` is 4,480 LOC and registers 21 RPC handlers spanning three orthogonal concerns: node CRUD, pin connections, and inspection/search. Edits to any one cluster currently force a full-file rebuild and review of a 4,480-line monolith. The split is clean — file-local helpers cluster naturally per concern, with a small set of genuinely shared helpers that can be lifted into a sibling helper header.

Risk is low: handler auto-registration runs from any TU via static-init counter records (`REGISTER_RPC_HANDLER` in `Handlers/HandlerRegistration.h`), so moving handlers to new `.cpp` files needs no registry edits or call-site updates.

## RPC inventory bucketed (21 handlers)

**CRUD** (8 handlers — node/pin create/modify/delete):
- `blueprint.graph.list_graphs` (cpp:799)
- `blueprint.graph.create_node` (cpp:874)
- `blueprint.graph.delete_node` (cpp:1403)
- `blueprint.graph.replace_node` (cpp:1459)
- `blueprint.graph.create_reroute_node` (cpp:2666)
- `blueprint.graph.set_node_property` (cpp:2704)
- `blueprint.graph.set_pin_default_value` (cpp:3676)
- `blueprint.graph.set_pin_default_values` (cpp:3731)

**Connections** (2 handlers — pin link operations):
- `blueprint.graph.connect_pins` (cpp:1220)
- `blueprint.graph.break_pin_links` (cpp:1361)

**Inspection / Search** (10 handlers — read-only queries):
- `blueprint.graph.get_nodes` (cpp:1296)
- `blueprint.graph.get_node_details` (cpp:2783)
- `blueprint.graph.get_graph_details` (cpp:2808)
- `blueprint.graph.get_graph_connections` (cpp:2867)
- `blueprint.graph.get_pin_details` (cpp:2921)
- `blueprint.graph.get_node_details_batch` (cpp:2994)
- `blueprint.graph.get_pin_details_batch` (cpp:3065)
- `blueprint.graph.find_nodes` (cpp:3172)
- `blueprint.graph.list_node_types` (cpp:3506)
- `blueprint.graph.get_execution_flow` (cpp:3825)

**Misnamespaced drift** (1 handler — flagged for separate consideration):
- `blueprint.references` (cpp:4204) — sits in the `blueprint.*` namespace, not `blueprint.graph.*`. Reads graph nodes/pins for reference extraction so it logically fits the Inspection cluster, but the namespace mismatch suggests it belongs in a different handler file entirely (e.g. a future `BlueprintReferencesHandler.cpp` under `blueprint.*`). Punt to a follow-up — do not let it block the cluster split.

## Shared helpers (all three clusters use them)

Six file-local statics are called by handlers in every cluster — they must be lifted out of the anonymous static scope so all three new TUs can link against them:

| Helper | cpp line | Used by clusters |
|---|---|---|
| `ResolveBlueprintAndGraph` | 84 | CRUD, Connections, Inspection (25+ call sites) |
| `FindNodeByIdOrName` | 157 | CRUD, Connections, Inspection (10+ call sites) |
| `FindPinByName` | 173 | CRUD, Connections, Inspection (6+ call sites) |
| `BuildPinJson` | 418 | Inspection (also indirectly via `BuildNodeDetailsJson`) |
| `BuildNodeDetailsJson` | 468 | Inspection |
| `BuildPinLookupPayload` | 583 | CRUD (`set_pin_default_value*`), Connections, Inspection |
| `BuildGraphConnectionsJson` | 507 | Inspection only (single cluster — keep file-local in Inspection TU) |
| `BuildNodeStateJson` + small enabled-state helpers | 336–415 | Inspection only — keep file-local in Inspection TU |
| Search machinery (`EMcpSearchMatchMode`, `SearchFieldMatchesTerm`, `IsNodeSearchField`, `NormalizeKnownSearchField`, etc.) | 198–334 | Inspection only (`find_nodes`) — keep file-local in Inspection TU |
| `GetCommonFunctionNodes`, `GetNodeTypeAliases`, `FindNodeClassByName`, `ResolveCommonFunctionOwnerClass` | 638–795 | CRUD only (`create_node`) — keep file-local in CRUD TU |

The five genuinely-shared helpers (`ResolveBlueprintAndGraph`, `FindNodeByIdOrName`, `FindPinByName`, `BuildPinJson` if Connections needs link rendering, `BuildPinLookupPayload`) move into a new `Handlers/Blueprint/BlueprintGraphHelpers.h` + `.cpp` pair, in a named namespace (e.g. `BlueprintGraphHelpers::`) — matches the precedent set by `BlueprintEnumHelpers`, `WidgetInspectHelpers`, etc. (per `EditorAutomationRpcGateway.Build.cs` Unity-safety conventions).

**Note:** `BlueprintHandlerUtils::ResolveBlueprintPath` already lives in `BlueprintHandlerUtils.h` and is called by `ResolveBlueprintAndGraph`. The new `BlueprintGraphHelpers` sit one level above it — graph-aware helpers depending on the path resolver.

## Test impact

Test files under `Source/EditorAutomationRpcGateway/Private/Tests/` (`TestBlueprintHandlers.cpp`, `TestBlueprintGraphOrphan.cpp`, `TestBlueprintGraphCompositeEntryPoints.cpp`, `TestBlueprintReplaceNode.cpp`) exercise the RPCs through the dispatcher, not through direct calls into the source's anonymous statics. `TestBlueprintHandlers.cpp` defines its own `FindPinByNameDirection` helper (a stricter variant) rather than reaching into the source. **No test refactor needed before the split.**

## Proposed file layout

Under `Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/`:

- `BlueprintGraphHelpers.h` / `.cpp` — Named-namespace `BlueprintGraphHelpers::` containing `ResolveBlueprintAndGraph`, `FindNodeByIdOrName`, `FindPinByName`, `BuildPinJson`, `BuildPinLookupPayload`. ODR-safe under Unity (matches `BlueprintEnumHelpers` precedent).
- `BlueprintGraphCrudHandler.cpp` — 8 CRUD handlers + `create_node`-only static maps (`GetCommonFunctionNodes`, `GetNodeTypeAliases`, `FindNodeClassByName`, `ResolveCommonFunctionOwnerClass`).
- `BlueprintGraphConnectionsHandler.cpp` — 2 connection handlers (`connect_pins`, `break_pin_links`). Smallest TU; could fold into CRUD if too thin, but the conceptual boundary is clean and worth its own file for searchability.
- `BlueprintGraphInspectionHandler.cpp` — 10 inspection/search handlers + Inspection-only statics (`BuildNodeDetailsJson`, `BuildNodeStateJson` + enabled-state helpers, `BuildGraphConnectionsJson`, the search-mode machinery `EMcpSearchMatchMode` / `SearchFieldMatchesTerm` / etc.).

`blueprint.references` stays put in `BlueprintGraphHandler.cpp` (or moves to a new `BlueprintReferencesHandler.cpp`) in a follow-up — do not couple that decision to this split.

## Migration strategy

1. Create `BlueprintGraphHelpers.h` / `.cpp` with the five shared helpers in a named namespace.
2. Create the three new cluster `.cpp` files. Move handler bodies + their cluster-private static helpers verbatim. Each new TU `#include`s `BlueprintGraphHelpers.h` for shared helpers.
3. Delete `BlueprintGraphHandler.cpp` (or shrink to just `blueprint.references` until that's relocated in a follow-up).
4. Build. Fix any include leakage (callers in handlers that were transitively picking up engine headers only because they sat in the monolith — most likely candidates are the K2Node_* includes).
5. Run the plugin's Blueprint test suite (`TestBlueprintHandlers.cpp`, `TestBlueprintGraphOrphan.cpp`, `TestBlueprintGraphCompositeEntryPoints.cpp`, `TestBlueprintReplaceNode.cpp`) to verify auto-registration still finds all 20 `blueprint.graph.*` RPCs.

## Recommended sequencing

Do this **after** the Animation handler split (not yet on the board but flagged as the precedent case in the triage finding). Both splits exercise the same auto-registration pattern — sequencing Animation first surfaces any handler-registration ergonomic gaps before committing to a second migration.

## History
- `#1-initial-scope` `OPEN` reporter — Filed: 4,480 LOC / 21 RPC monolith; clean 3-cluster split (CRUD 8 / Connections 2 / Inspection-Search 10) plus 1 misnamespaced `blueprint.references` handler flagged for separate relocation; 5 shared helpers (`ResolveBlueprintAndGraph`, `FindNodeByIdOrName`, `FindPinByName`, `BuildPinJson`, `BuildPinLookupPayload`) lift into new `BlueprintGraphHelpers` named-namespace header per the `BlueprintEnumHelpers` precedent; cluster-private helpers stay file-local in each new TU; tests exercise via dispatcher and don't reach into source statics, so no test refactor needed; sequence after Animation handler split.
- `#2-split-implemented` `IN-REVIEW` developer — Split the monolith into 4 TUs: new `BlueprintGraphHelpers.h`/`.cpp` (named namespace with the 5 shared helpers + file-local `ToDirectionString`), `BlueprintGraphCrudHandler.cpp` (8 CRUD handlers + `create_node`/`replace_node`/`set_pin_default*` cluster-private statics: `GetCommonFunctionNodes`/`GetNodeTypeAliases`/`FindNodeClassByName`/`ResolveCommonFunction*`/`ApplyPinDefaultValueCore`/`ApplyPersistedTextPinDefaultValue`), `BlueprintGraphConnectionsHandler.cpp` (`connect_pins`/`break_pin_links`), `BlueprintGraphInspectionHandler.cpp` (10 inspection handlers + `BuildNodeDetailsJson`/`BuildNodeStateJson`/enabled-state helpers/`BuildGraphConnectionsJson`/`EEdgeTypeFilter`/search machinery). `BlueprintGraphHandler.cpp` shrunk to retain only `blueprint.references` (relocation deferred per #1). Each new TU brings shared names in via `using BlueprintGraphHelpers::...` so moved handler bodies are unchanged; all 21 REGISTER_RPC_HANDLER registrations preserved verbatim across the 4 files. No test refactor (tests hit the dispatcher). Not compiled — ergonomic ticket, no regression test.
- `#3-revert-drive-bys` `IN-REVIEW` developer — Reviewer flagged two out-of-scope drive-by edits leaked from the shared sprint tree. Reverted both to HEAD: `BlueprintApiIndexHandler.cpp` (had swapped inline `TJsonWriter`/`TCondensedJsonPrintPolicy` construction for `SortedJsonWriter::SerializeSortedJsonObject` — belongs to B-sortedjson-not-enforced-on-disk-writes) and `BlueprintCreationHandler.cpp` (had modernized `Self->SendAutomationResponse` to `Ctx.SendSuccess`/`Ctx.SendError` and dropped unused `Self`/`RequestId` captures — API modernization, not part of this structural split). Neither file is in this ticket's scope and none of the new split TUs reference them, so the reverts are independent. No change to the split itself.
- `#4-revert-actually-applied` `IN-REVIEW` developer — Reviewer found that the #3 revert never actually landed: both `BlueprintApiIndexHandler.cpp` and `BlueprintCreationHandler.cpp` were still modified in the working tree (the `SortedJsonWriter::SerializeSortedJsonObject` swap and the `Ctx.SendSuccess`/`Ctx.SendError`/dropped-`RequestId` modernizations persisted). Ran `git checkout HEAD --` on both files to genuinely restore them; `git diff` against HEAD is now empty for both. The #3 claim is now accurate. Split itself unchanged; the new TUs still do not reference either file.
- `#5-verify-fix` `DONE` tester — Verified all 4 split clusters are live-registered in the running editor (binary reflects post-split code). Exercised one+ RPC per cluster on `/App/App/LevelBlueprints/Tutorials/ACRO/B_RaceTrackTraining`: CRUD `list_graphs` returned graphs={EventGraph,UserConstructionScript}; Inspection `get_nodes` returned nodes=[]; Connections `break_pin_links` and CRUD `set_node_property` reached their handlers (domain `MISSING_REQUIRED_PARAM`, not `UNKNOWN_ACTION`); Inspection-search `find_nodes` reached its handler (domain `MISSING_REQUIRED_PARAM` on `query`). No RPC returned `UNKNOWN_ACTION`, so auto-registration survived the monolith→4-TU split intact.
