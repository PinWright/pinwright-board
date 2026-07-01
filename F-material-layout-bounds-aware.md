---
id: F-material-layout-bounds-aware
title: "Bounds-aware material-graph layout (FMGIRLayoutEngine)"
status: OPEN
severity: Medium
category: feature
tags: [layout, material, mgir, node-size]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Bounds-aware material-graph layout (FMGIRLayoutEngine)

`FMGIRLayoutEngine` places material expressions on a **fixed depth-lane
grid** with no node-size awareness:
`Source/PinWright/Private/MGIR/MGIRLayoutEngine.cpp` (lane placement at lines
49-69 — `HorizontalSpacing` line 65, `VerticalSpacing` line 66; unpositioned
expressions get `OriginX + Depth * HorizontalSpacing` /
`OriginY + Lane * VerticalSpacing`). Because the spacing constants are fixed
and ignore actual expression dimensions, wide or tall expressions (e.g.
material functions, large parameter nodes) overlap their neighbors.

## What it should do

Place expressions using real measured bounds instead of a constant grid:
- Real **column widths** derived from the widest expression in each depth
  lane, and real **row heights** per row, using the metrics-core node-size
  estimator's `UMaterialExpression` adapter (`F-graph-layout-metrics-core`).
- A final **overlap-resolution sweep** — a shared version of BPIR's
  `FormatY` collision approach (`FNodeLayoutEngine::FormatY`,
  `Source/PinWright/Private/Compiler/NodeLayoutEngine.cpp:1268`) — to push
  apart any residual collisions after lane placement.

## Proposed approach

- Replace the fixed `HorizontalSpacing`/`VerticalSpacing` stepping in
  `LayoutExpressions` with per-lane width / per-row height computed from the
  node-size estimator.
- Reuse the shared collision-resolution sweep extracted alongside / from
  `FNodeLayoutEngine::FormatY` rather than duplicating it.
- **Regression test:** run auto-layout on a real material graph with
  wide/tall expressions, then call `FGraphLayoutMetrics` and assert
  `overlap == 0` (no bounding-box intersection) and an improved combined
  score versus the fixed-grid baseline.

## Severity justification

**Medium.** Soft blocker: material auto-layout is doable today but produces
overlapping output on non-trivial graphs, requiring manual cleanup or a
documented workaround. No crash, no data corruption. Blocked by
`F-graph-layout-metrics-core` (needs the node-size estimator and the metrics
util for both the placement math and the regression assertion).

## History
- `#1-initial-spec` `OPEN` reporter — FMGIRLayoutEngine (MGIRLayoutEngine.cpp:49-69) lays expressions on a fixed depth-lane grid with no node-size awareness, so wide/tall expressions overlap; replace with bounds-aware placement (metrics-core estimator) + a FormatY-style overlap sweep, verified by FGraphLayoutMetrics asserting no overlap.
