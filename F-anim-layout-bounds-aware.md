---
id: F-anim-layout-bounds-aware
title: "Bounds-aware anim-graph layout (FAGIRLayoutEngine)"
status: WONTFIX
severity: Medium
category: feature
tags: [layout, anim, agir, node-size, state-machine]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Bounds-aware anim-graph layout (FAGIRLayoutEngine)

`FAGIRLayoutEngine` lays out anim-graph nodes without node-size awareness:
`Source/PinWright/Private/AGIR/AGIRLayoutEngine.cpp` (`FAGIRLayoutEngine::Layout`
from line 146; spec-cited body ~193-273). The engine already **recursively
descends into state-machine inner graphs and their nested children**
(`CollectStateMachineInnerGraphs` at line 62; recursion via
`InnerGraph->GetAllChildrenGraphs` around line 85; the collection loop at
lines 177-185), so any bounds-aware change must cover those inner graphs too,
not just the top-level anim graph. Without measured bounds, wide/tall anim
nodes (blend spaces, large state nodes, layered blends) overlap.

## What it should do

- Place anim-graph nodes using real measured bounds (column widths / row
  heights) from the metrics-core node-size estimator's anim-node adapter
  (`F-graph-layout-metrics-core`), replacing fixed-grid stepping.
- Apply the same bounds-aware placement **recursively** to every collected
  state-machine inner graph, preserving the existing recursive collection.
- Finish with a shared overlap-resolution sweep (the `FormatY`-style
  collision approach, `FNodeLayoutEngine::FormatY`,
  `Source/PinWright/Private/Compiler/NodeLayoutEngine.cpp:1268`).

## Proposed approach

- Thread the node-size estimator through `FAGIRLayoutEngine::Layout` and the
  inner-graph layout path so both top-level and state-machine inner graphs
  use measured dimensions.
- Reuse the shared collision-resolution sweep; do not duplicate it.
- **Regression test:** run auto-layout on a real anim graph that includes a
  state machine with inner-graph nodes, then call `FGraphLayoutMetrics` on
  both the outer graph and an inner graph and assert `overlap == 0` and an
  improved combined score versus the baseline.

## Severity justification

**Medium.** Soft blocker: anim auto-layout works but overlaps on non-trivial
graphs (the state-machine inner-graph case is the most likely offender), and
cleanup is manual. No crash, no data corruption. Blocked by
`F-graph-layout-metrics-core` for the estimator and the metrics assertion.

## History
- `#1-initial-spec` `OPEN` reporter — FAGIRLayoutEngine (AGIRLayoutEngine.cpp:193-273) has no node-size awareness and recurses into state-machine inner graphs; make placement bounds-aware (metrics-core estimator) across outer and inner graphs + a FormatY-style overlap sweep, verified by FGraphLayoutMetrics asserting no overlap.
- `#2-repointed-to-graph-layout-core` `OPEN` reporter — Gap analysis 2026-09-30: re-pointed to F-graph-layout-core, the shared layered layout core whose graph-type adapter replaces this fixed-grid engine (size-aware placement, pin-aligned Y, barycenter ordering, overlap-free packing). Implement this ticket as that adapter rather than as a standalone engine; added F-graph-layout-core to blockedBy.
- `#3-stale-sweep-engine-replaced` `WONTFIX` developer — Superseded by `F-graph-layout-core` (DONE). Plugin commit `0a3b3acc` deleted `AGIR/AGIRLayoutEngine.{h,cpp}` (the fixed-grid `FAGIRLayoutEngine` this ticket asks to make bounds-aware); AGIR compiles now call `PwGraphLayout::ArrangeAnimBlueprint` (`AGIRCompiler.cpp:1749`), which runs the size-aware layered core over every anim graph and walks nested state-machine graphs (`Layout/PwGraphLayoutEdGraph.cpp:34-64`, `:235-265`), with measured node sizes from `c343fa0d` and the `PoseChainGrowsLeftFromResult` adapter test cited in `F-graph-layout-core` `#11`. The engine this ticket targets no longer exists; any remaining anim-layout defect should be filed against the shared core.
