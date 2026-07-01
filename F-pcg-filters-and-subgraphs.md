---
id: F-pcg-filters-and-subgraphs
title: "PCG slope/noise filters, subgraphs, self-pruning helpers"
status: DONE
severity: Medium
category: feature
tags: [pcg, procedural-content-generation, graph-authoring, ergonomic, follow-on]
---

# Typed PCG helpers beyond plain spawning: filters, subgraphs, self-pruning

Follow-on to [`F-pcg-graph-authoring`](F-pcg-graph-authoring.md). Once
the base `pcg.*` namespace lands (UPCGGraph asset creation, primitive
node spawning, edge wiring, parameter writes, a verified single
mesh-spawner round trip), this ticket adds the **typed helpers** that
make the PCG surface competitive with Microsoft CoPilot's claimed PCG
authoring differentiators.

PCG (Procedural Content Generation) is UE5's native node-graph system
for procedural worlds (`/Script/PCG.PCGGraph` assets, edited via the
PCG Graph editor). Stock node classes live under `UPCGSettings`
descendants: `UPCGDensityFilterSettings`, `UPCGAttributeFilterSettings`,
`UPCGSurfaceSamplerSettings`, `UPCGStaticMeshSpawnerSettings`,
`UPCGSubgraphSettings`, `UPCGSelfPruningSettings`, etc. Each node in
the graph is a `UPCGNode` wrapping a settings instance.

**Proposed helpers (all on `pcg.*`):**

1. `pcg.add_slope_filter` — spawns a slope/density filter node
   (`UPCGDensityFilterSettings` or the slope-specific subclass) with
   `minSlope`, `maxSlope`, `densityThreshold` typed params. The
   primitive `pcg.add_node` from F-pcg-graph-authoring can also create
   these, but callers then need 3+ property writes per node. The typed
   helper collapses to one round trip.

2. `pcg.add_noise_filter` — spawns a noise-based filter (Perlin /
   Voronoi / Worley depending on the underlying settings class), with
   typed `noiseType`, `frequency`, `amplitude`, `seed`, `threshold`
   params.

3. `pcg.add_subgraph` — inserts a `UPCGSubgraphNode` (or whatever
   wraps `UPCGSubgraphSettings`) referencing another `UPCGGraph` asset
   by path. Validates the referenced asset exists and is a
   `UPCGGraph`; rejects with `INVALID_SUBGRAPH_ASSET` otherwise.
   Resolves the asset reference via the standard
   `Utils/AssetUtils::ResolveAssetByPath` path.

4. `pcg.set_self_pruning_settings` — writes the typical
   `UPCGSelfPruningSettings` knobs (`pruningRadius`, `density`,
   `pruneMode`) on an already-existing self-pruning node. Mirrors the
   shape of `material.set_parameter` etc.

**Why typed helpers and not just "use `pcg.add_node` + property writes":**

Pairs with the same argument made on
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md):
generic class-by-name node spawning works mechanically, but every
typed knob the caller wants becomes a separate `set_property` round
trip. For PCG, where one graph commonly has 10-30 nodes and most
nodes have 3-8 meaningful params each, the imperative-only path is
20-100+ round trips per graph. Typed helpers for the high-frequency
node classes (filters, subgraphs, self-pruning, the spawner from
F-pcg-graph-authoring) collapse the common cases to ~one call per
node.

**Use cases blocked until both tickets land:**

1. Authoring a "foliage scatter on slopes < 30° + noise variation"
   PCG graph (the canonical PCG demo) imperatively — requires slope
   filter + noise filter + spawner + self-pruning, all of which are
   on the typed helper list.
2. Composing PCG graphs from sub-pieces — without `add_subgraph`,
   callers must duplicate node trees across graphs.
3. Building density-controlled vegetation passes — self-pruning is
   what turns a uniform point cloud into varied placement.

**Dependency:** Blocked by
[`F-pcg-graph-authoring`](F-pcg-graph-authoring.md). That ticket
must land first (UPCGGraph asset CRUD, `pcg.add_node` primitive,
`pcg.connect` edge wiring, `pcg.set_node_property`, plus a verified
end-to-end single mesh-spawner round trip) before this ticket's typed
helpers have a substrate to build on. Filed now so the design intent
("typed helpers for the high-frequency PCG node classes") is captured
alongside the base authoring ticket; defer implementation until the
base is green.

**Cross-ref:** Same ergonomic argument as the AGIR / animation
authoring tickets — generic node-by-class creators are correct but
verbose; typed helpers for the high-frequency cases are the
ergonomic capstone. PCG also benefits from a text-IR (PCGIR?) but
that's a much larger separate ticket; the typed-helper layer is the
pragmatic near-term win.

## History
- `#1-filed-as-follow-on` `OPEN` reporter — Filed as the typed-helper follow-on to F-pcg-graph-authoring. Once the base pcg.* namespace (asset CRUD + primitive node spawning + edge wiring + a verified single mesh-spawner case) lands, this ticket adds `pcg.add_slope_filter`, `pcg.add_noise_filter`, `pcg.add_subgraph` (UPCGSubgraphNode referencing another UPCGGraph asset), and `pcg.set_self_pruning_settings` (radius/density/mode). Same ergonomic argument as F-anim-add-graph-node-generic: generic node-by-class creators work but collapse to N round trips per node; typed helpers for the high-frequency node classes reduce common PCG authoring (foliage on slopes, density-pruned vegetation, subgraph composition) to ~one call per node. Explicit dependency on F-pcg-graph-authoring noted; defer implementation until base authoring verified.
- `#2-implemented-typed-helpers` `IN-REVIEW` developer — Added `pcg.add_slope_filter` (UPCGNormalToDensitySettings), `pcg.add_noise_filter` (UPCGSpatialNoiseSettings for kind=spatial, UPCGAttributeNoiseSettings for kind=attribute), `pcg.add_subgraph` (UPCGSubgraphSettings::SetSubgraph; rejects recursion with RECURSIVE_SUBGRAPH and non-UPCGGraphInterface assets with INVALID_SUBGRAPH_ASSET), `pcg.set_self_pruning_settings` (writes nested Parameters struct: PruningType/RadiusSimilarityFactor/bRandomizedPruning; rejects wrong-type nodes). All four in `Handlers/PCG/` under the same `__has_include("PCGGraph.h")` guard as the core PCG handlers. Ticket's class-name guesses corrected: there is no slope-specific filter class — NormalToDensity is the canonical UE 5.6 analog. Tests in `Tests/PCG/PCGTypedHelpersTests.cpp` cover strength apply + Voronoi mode + recursive rejection + wrong-type rejection; counterfactual: reverting strength apply in add_slope_filter makes the property default-construct to 1.0 and the test fails.
- `#3-verify-fix` `DONE` tester — Verified: created temp UPCGGraph at /Game/App/UI/Test/W_McpVerifyTemp_F_pcg_filters_and_subgraphs, then ran all four typed helpers. pcg.add_slope_filter -> nodeId NormalToDensity_0 (UPCGNormalToDensitySettings); pcg.add_noise_filter kind=spatial -> Spatial Noise_0 (UPCGSpatialNoiseSettings); pcg.add_subgraph with self-ref returned RECURSIVE_SUBGRAPH, with /Engine/EngineMaterials/DefaultMaterial returned INVALID_SUBGRAPH_ASSET; pcg.set_self_pruning_settings on SelfPruning_0 with pruningType=LargeToSmall/factor=0.42/bRandomizedPruning=true succeeded, on NormalToDensity_0 returned WRONG_NODE_TYPE. pcg.inspect confirmed all three node classes present. Temp graph deleted. Note: pcg namespace wiki overlay incorrectly documents the subgraph arg as `subgraphAssetPath`; the actual schema (and handler) uses `subgraphAsset` — wiki overlay typo, separate ticket-worthy.
