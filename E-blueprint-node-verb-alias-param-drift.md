---
id: E-blueprint-node-verb-alias-param-drift
title: "blueprint.add_node / blueprint.connect_pins (top-level) duplicate blueprint.graph.create_node / blueprint.graph.connect_pins with DIVERGENT param names, and no overlay reconciles them — forcing a double doc-read per verb pair"
status: OPEN
severity: Low
category: ergonomic
tags: [near-duplicate-verb-descriptions, docs, blueprint, discoverability, param-drift, wiki]
encounters: 1
lastSeen: 2026-07-04T16:19:56.0000000Z
---

# Two node-authoring verb pairs exist across `blueprint.*` and `blueprint.graph.*`, with divergent param vocabularies and no cross-reference

The plugin registers **two parallel node-authoring verb pairs** that do the same
job but live in different namespaces and use **different parameter names**:

- **Create a node:**
  - `blueprint.add_node` (top-level `blueprint` namespace,
    `BlueprintComponentHandler.cpp:733`) — takes `functionName` / `variableName`
    + `posX` / `posY`.
  - `blueprint.graph.create_node` (graph namespace,
    `BlueprintGraphCrudHandler.cpp`) — takes `memberName` (+ `target`) + `x` / `y`,
    and echoes `nodeId`.
- **Connect two pins:**
  - `blueprint.connect_pins` (top-level `blueprint` namespace,
    `BlueprintComponentHandler.cpp:1018`) — requires
    `sourceNodeGuid` / `targetNodeGuid` / `sourcePinName` / `targetPinName`.
  - `blueprint.graph.connect_pins` (graph namespace,
    `BlueprintGraphConnectionsHandler.cpp`) — takes
    `fromNodeId` / `fromPinName` / `toNodeId` / `toPinName`.

So the same intent ("add a node", "wire two pins") is served by two verbs whose
parameter names disagree on **every** field: `sourceNodeGuid` vs `fromNodeId`,
`targetNodeGuid` vs `toNodeId`, `posX`/`posY` vs `x`/`y`, `functionName` vs
`memberName`. There is no cross-link on either overlay page saying which is
canonical / preferred, or that the two are near-aliases, so a caller who lands on
one page has no signal that the other exists — or which one the rest of the graph
tooling (`get_nodes`, `get_graph_connections`, `compile_bpir`'s `createdNodes`,
`add_event`'s connect-me-next flow) speaks. The graph tooling all returns / consumes
`nodeId`-style GUIDs, so `blueprint.graph.*` is the one that composes with the
rest of the surface; `blueprint.add_node`/`blueprint.connect_pins` are the odd
`sourceNodeGuid`/`posX` twins.

## Friction observed (this task)

Clean realism task (focus `blueprint.connect_pins`, namespace `blueprint`,
outcome ergo — `/Game/BP_SpawnBeacon` BeginPlay→PrintString→Set(bInitialized)
authored, wired, compiled `UpToDate` 0-errors, saved; 16 RPCs all `ok`/non-error,
zero retries). The wiki-nav phase read **both pages of both pairs** before the
execute phase committed to the `blueprint.graph.*` forms:

- `blueprint.add_node.md` AND `blueprint.graph.create_node.md`
- `blueprint.connect_pins.md` AND `blueprint.graph.connect_pins.md`

The CallAnalyzer's call-trace finding (verbatim): *"Two methods do the same job
with DIVERGENT parameter names (sourceNodeGuid vs fromNodeId; posX/posY vs x/y;
functionName vs memberName). The agent ultimately used the blueprint.graph.*
variants."* The agent picked the right (composable) variant on the first execute —
so this is pure **discovery friction** (a double doc-read to disambiguate), not a
functional bug or a wrong call.

## What it should do / how to fix (docs-first; NAMES the overlay pages)

The overlay edit is a downstream wiki process, not this audit's job — naming the
pages and the reconciliation is the deliverable. Follow the accepted
overlay-cross-reference pattern already proven by
`E-asset-search-vs-search-assets-overlap` and
`E-anim-blueprint-create-two-methods-discovery` (reciprocal `##` sections + a
`WikiHandler::RenderPage` regression test):

- `Docs/wiki-src/blueprint.md` — add a short "node-authoring verbs: `blueprint.*`
  vs `blueprint.graph.*`" note that states plainly the top-level
  `blueprint.add_node` / `blueprint.connect_pins` are the legacy twins of
  `blueprint.graph.create_node` / `blueprint.graph.connect_pins`, name the
  param-name mapping (`sourceNodeGuid`↔`fromNodeId`, `targetNodeGuid`↔`toNodeId`,
  `posX/posY`↔`x/y`, `functionName`↔`memberName`), and point at the
  `blueprint.graph.*` pair as canonical/preferred because it composes with the
  rest of the graph tooling (`get_nodes`/`get_graph_connections`/`compile_bpir`
  all speak `nodeId`).
- `Docs/wiki-src/blueprint.graph.md` — a reciprocal one-liner on the
  `create_node` / `connect_pins` sections noting the top-level `blueprint.add_node`
  / `blueprint.connect_pins` aliases and their `sourceNodeGuid`/`posX` param names,
  so a caller landing on either page is routed without reading its twin.

A cheaper structural option (out of scope for this docs-tagged ticket) would be to
converge the param vocabularies (accept `fromNodeId`/`x`/`y` as aliases on the
top-level verbs, or vice-versa) so the two verbs are drop-in interchangeable — but
the doc note unblocks callers now without any handler change.

## Distinct from

- `E-insert-undo-pairing-undocumented` (OPEN, same `near-duplicate-verb-descriptions`
  family tag) — that ticket is the `insert_*`/`undo_last_*` pairing + "code vs BPIR"
  distinction (byte-identical *summaries*, same namespace); this one is a
  *param-name divergence across two namespaces* (`blueprint.*` vs
  `blueprint.graph.*`) on the node create/connect verbs. Different methods,
  different remedy (map the param vocab vs define the code/BPIR terms). Family
  siblings, not duplicates.
- `E-asset-search-vs-search-assets-overlap` / `E-anim-blueprint-create-two-methods-discovery`
  (IN-REVIEW) — same "overlapping verbs, overlay never reconciles" shape but on the
  `asset.*` and `animation.*` namespaces, distinct methods. This is the
  `blueprint.*` node-authoring instance of the family.
- `E-add-event-no-node-id-echo` (OPEN, judge-filed for this same task) — the
  `add_event` missing-`nodeId` echo; a response-shape gap on a different method.

## Evidence

Call log: the wiki-nav block read `blueprint.add_node.md`,
`blueprint.graph.create_node.md`, `blueprint.connect_pins.md`, and
`blueprint.graph.connect_pins.md` (four pages across two verb pairs) before the
execute phase used `blueprint.graph.create_node` (×2) and `blueprint.graph.connect_pins`
(×2). Source-confirmed the four distinct registered handlers and their divergent
required params: `blueprint.connect_pins` REQ `sourceNodeGuid`/`targetNodeGuid`
(`BlueprintComponentHandler.cpp:1021-1022`) vs `blueprint.graph.connect_pins`
`fromNodeId`/`toNodeId`; `blueprint.add_node` `functionName`/`posX`/`posY`
(`BlueprintComponentHandler.cpp:733`) vs `blueprint.graph.create_node`
`memberName`/`x`/`y`. Neither `Docs/wiki-src/blueprint.md` nor
`Docs/wiki-src/blueprint.graph.md` cross-references the twin pair.

severity rationale: impact=docs/discoverability (a double doc-read to disambiguate;
the agent still picked the composable variant on the first execute, no failed call)
× reach=every-session (node create/connect is a hot BP-authoring path) -> Low.
Held at Low (not bumped) to match the sibling overlap tickets
(`E-asset-search-vs-search-assets-overlap`, `E-anim-blueprint-create-two-methods-discovery`,
both Low) — the friction is a cheap read cost, not a wrong call.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle-audit of a clean realism BP-wiring task (focus `blueprint.connect_pins`, namespace `blueprint`, outcome ergo, 16 RPCs all `ok`/non-error, zero retries) that built `/Game/BP_SpawnBeacon` (BeginPlay→PrintString→Set bInitialized, wired + compiled `UpToDate` + saved). Discovery friction: the wiki-nav phase read BOTH pages of BOTH node-authoring verb pairs — `blueprint.add_node.md` + `blueprint.graph.create_node.md`, and `blueprint.connect_pins.md` + `blueprint.graph.connect_pins.md` — before committing to the `blueprint.graph.*` forms, because the top-level `blueprint.add_node`/`blueprint.connect_pins` duplicate the `blueprint.graph.create_node`/`blueprint.graph.connect_pins` job with DIVERGENT param names (`sourceNodeGuid`/`targetNodeGuid` vs `fromNodeId`/`toNodeId`; `posX`/`posY` vs `x`/`y`; `functionName` vs `memberName`) and no overlay cross-links them. Source-confirmed the four distinct handlers + params (`BlueprintComponentHandler.cpp:733,1018-1022` for the top-level twins; graph twins in `BlueprintGraphCrudHandler`/`BlueprintGraphConnectionsHandler`). Neither `blueprint.md` nor `blueprint.graph.md` reconciles the pair. Dedup: ripgrep across OPEN/closed — `E-blueprintgraph-handler-split` (DONE) is internal code-org only; `E-insert-undo-pairing-undocumented` is a different method set (insert/undo, identical summaries, same namespace) sharing only the family tag; the asset/anim overlap tickets are other namespaces. No ticket pairs `blueprint.connect_pins`/`add_node` with `blueprint.graph.*`. Proposed: reconcile the twin pairs in `Docs/wiki-src/blueprint.md` (+ reciprocal note on `blueprint.graph.md`) naming the param-name mapping and marking `blueprint.graph.*` canonical (it composes with the rest of the graph tooling). Docs-only; no handler change.
