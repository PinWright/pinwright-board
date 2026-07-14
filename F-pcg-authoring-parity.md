---
id: F-pcg-authoring-parity
title: "pcg: graph user-parameter CRUD (add/list/remove)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [pcg, authoring, graph-parameters]
---

# pcg: graph user-parameter CRUD (add/list/remove)

PinWright's `pcg` namespace can author a graph's node/edge structure (`create_graph`,
`add_node`, `connect_pins`, `remove_node`, typed filter/subgraph helpers) but has **no
handler for a graph's user parameters** — the overridable inputs shown in the PCG Graph
editor's "Graph Parameters" panel, stored on `UPCGGraph` as an `FInstancedPropertyBag`
(`GetUserParametersStruct()` / `UpdateUserParametersStruct()`, UE 5.7). Today an agent can
only add/read/remove them via `python.execute`.

Add the asset-authoring CRUD surface for that bag, mirroring the existing imperative
`pcg.*` handlers (load graph by path, mutate, `MarkPackageDirty`):

- `pcg.add_graph_parameter(graphPath, name, type, [value])` — upsert a scalar parameter
  (`bool|int|int64|float|double|name|string`) with an optional initial value.
- `pcg.list_graph_parameters(graphPath)` — list parameters as `{name, type, value?}`.
- `pcg.remove_graph_parameter(graphPath, name)` — remove by name; `PARAMETER_NOT_FOUND` if absent.

**Acceptance:** an agent can add a typed parameter with a value, list it back with its
type, and remove it; a round-trip is verified in-code against the graph's own
`FInstancedPropertyBag` (not `python.execute`).

## Scope history — split from an audit umbrella

This ticket was originally "audit pcg vs UE 5.8 PCGToolset (31 tools) and close authoring
gaps," bundling (1) a non-code gap-table deliverable "attached via IN-REVIEW note," (2)
three separate RPC families, and (3) an end-to-end generate+readback demo. That is a
mini-epic, not a single verified-green fix, and it contradicts the board's established
discrete-per-capability PCG cadence (`F-pcg-core-graph` split typed helpers off as YAGNI →
`F-pcg-filters-and-subgraphs`; `F-pcg-decompile-ir` split decompile off). The "31-tool
parity" premise is also inflated: `C:\UE_5.8\...\PCGToolset` is Epic's agent **skill pack**
(biome/instancing recipe skills over a `UPCGToolset` library), not 31 discrete graph-authoring
RPCs — the real gaps collapse to the three the reporter named. So the count-based gap-table
audit is dropped, and the code work is split:

- **This ticket** — graph user-parameter CRUD (the cleanest, synchronously-verifiable slice).
- `F-pcg-generate-readback` — `pcg.generate`: trigger `UPCGComponent` generation on a placed
  actor via the async completion-token seam and read back point counts (highest value; today
  only `python.execute` does this).
- `F-pcg-set-node-property` — generic per-node settings editor (Epic `UpdateNode`-style).
  Low priority: `property.set` + the shipped typed helpers already partially cover it.

## History
- `#1-pcg-3x-gap` `OPEN` reporter — UE 5.8.0 shipped PCGToolset with 31 tools vs our 10 + PCGIR; deltas unaudited. Audit first, then close authoring-relevant gaps (generation trigger/readback, node settings, graph params).
- `#2-reword-to-graph-param-crud` `IN-REVIEW` developer — REWORD + split. Gap verified real (exactly 10 `pcg.*` handlers; no graph-parameter CRUD, no generate/readback, no generic per-node settings). Dropped the non-code 31-tool gap-table audit umbrella (PCGToolset is Epic's agent skill pack, not 31 authoring RPCs) and split the code into discrete capability tickets per the board's PCG precedent. This ticket is now scoped to graph user-parameter CRUD (`pcg.add_graph_parameter` / `list_graph_parameters` / `remove_graph_parameter`) on `UPCGGraph`'s `FInstancedPropertyBag`; generate+readback → new `F-pcg-generate-readback`, generic per-node settings → new `F-pcg-set-node-property`. Shipped `Handlers/PCG/PCGGraphParameters.cpp` (3 handlers add/list/remove via `UpdateUserParametersStruct` + `FInstancedPropertyBag`, scalar types bool|int|int64|float|double|name|string, reused error codes GRAPH_NOT_FOUND/INVALID_NAME/INVALID_PARAM_TYPE/ADD_FAILED/SET_FAILED/PARAMETER_NOT_FOUND/REMOVE_FAILED) + wiki overlay `Docs/wiki-src/pcg.md`. Regression test `PinWright.pcg.graph_parameter.AddListRemoveRoundTrip` (`Tests/PCG/PCGGraphParametersTests.cpp`) drives the real handlers through the dispatch-capture seam and asserts against the graph's own bag. Compiles clean; test passes; differential stash-verified (pre-fix `Result={Fail}` on the 'add_graph_parameter registered' assertion).
