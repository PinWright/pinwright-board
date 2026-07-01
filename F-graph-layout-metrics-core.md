---
id: F-graph-layout-metrics-core
title: "Reusable engine-agnostic graph layout-quality metrics util (FGraphLayoutMetrics) + node-size adapter seam"
status: IN-REVIEW
severity: High
category: feature
tags: [layout, metrics, node-size, bpir]
blockedBy: []
---

# Reusable engine-agnostic graph layout-quality metrics util + node-size adapter seam

There is no objective measure of node-graph layout quality, so poor layout
ships silently on every MCP-generated graph. `B-node-layout-poor` was closed
(DONE) on a trivial 4-node check (`W_McpVerifyTemp`, Event Construct →
GetName → PrintString → SetVisibility) — that verification proves a clean
toy graph stays clean, but says nothing about real graphs, which still lay
out badly. Without a metric we cannot detect the regression, gate it in
tests, or compare one layout engine against another.

This ticket is the **keystone code util** that the dependent cluster
consumes (`F-graph-layout-report-rpc`, `F-blueprint-auto-layout-rpc`,
`F-anim-auto-layout-rpc`, `F-controlrig-auto-layout-rpc`,
`F-material-layout-bounds-aware`, `F-anim-layout-bounds-aware`,
`F-controlrig-layout-bounds-aware`). Each of those feeds geometry into the
util or asks the node-size seam for sizes; none needs the threshold
calibration to begin. The threshold-derivation work and the
measurement-seeded follow-up tickets are split out (see "Scope" below) so
this ticket is a bounded, unit-testable unit of work.

## What this ticket delivers (in scope)

Two reusable building blocks, neither of which exists today:

**(a) A pure `FGraphLayoutMetrics` util** — engine-agnostic, no editor RPC,
no asset deps, no node-type coupling. Input: `[{nodeId,x,y,w,h}]` plus edges
`[{from,to}]`. Output: sub-scores —
- **overlap** — pairwise bounding-box intersection area (absolute; any
  overlap is a hard fail → overlap sub-score 0 and a non-empty list of the
  overlapping node-id pairs),
- **spacing** — distribution of gaps between adjacent nodes,
- **straightness** — approximated from node-center alignment along edges,
- **edge-crossings** — approximated from segment crossings between
  node-center-to-node-center lines,
- **grid** — snap/alignment regularity,

and a combined **0-1 score**. Straightness and crossings are approximated
from node centers because pin coordinates and actual wire routing are not
exposed at this layer — document that limitation in the util. This is the
exact shape `F-graph-layout-report-rpc` feeds (`[{nodeId,x,y,w,h}]` + edges).

**(b) A node-size adapter seam.** BPIR already has a real Slate-measurement
estimator: `BpirLayout::EstimateNodeSize` (declared `PINWRIGHT_API` in
`Source/PinWright/Private/Compiler/NodeLayoutEngine.h:20`, impl at
`NodeLayoutEngine.cpp:259`; real measurement via `FSlateFontMeasure::Measure()`
at `NodeLayoutEngine.cpp:72`, char-count fallback at `NodeLayoutEngine.cpp:37-47`).
But it is `UEdGraphNode`/Blueprint-specific. The material, anim, and rig
layout engines have **no node-size awareness at all** — they place on fixed
grids (`MGIRLayoutEngine.cpp:65-66`, `AGIRLayoutEngine.cpp:269-270`,
`CRIRLayoutEngine.cpp:146-147`). Add a tiny engine-agnostic node-size adapter
interface plus the **Blueprint adapter** that wraps the existing
`EstimateNodeSize`, so every engine and report can ask "how big is this node"
through one shared seam. The **`UMaterialExpression` / anim-node / rig-node
adapter bodies** are delivered with their own bounds-aware engine tickets
(they need each engine's node types and the bounds-aware integration) — this
ticket lands the seam + Blueprint adapter so those tickets have something to
plug into.

## Proposed approach

- Add `FGraphLayoutMetrics` as a pure util under a new `Private/Layout/`
  (header + cpp), exported `PINWRIGHT_API`, unit-testable in isolation and
  callable from every layout engine, the report RPCs, and the bounds-aware
  engine work.
- Add a `INodeSizeAdapter`-style interface (one virtual: size of a node by
  opaque pointer) in the same `Private/Layout/` header, plus a Blueprint
  adapter that wraps `BpirLayout::EstimateNodeSize`. Reuse the existing
  Slate-measurement path; do not rewrite it.
- **Unit test (in scope):** score a clean synthetic graph (well-spaced, no
  overlap) high, and an overlapping synthetic graph low, asserting
  `overlap > 0` (overlap sub-score 0 + non-empty overlapping-pair list) on
  the bad one and a strictly higher combined score on the clean one.

## Scope — what is split out of this ticket

The original spec folded an open-ended threshold-calibration pass and two
measurement-seeded follow-up tickets into this unit. Those are **not** code
this util needs and **none of the 7 dependents block on them**, so they are
separated:

- **Calibration / threshold derivation** → `F-graph-layout-calibration`
  (deferred): score human-authored vs MCP corpora, derive the absolute
  overlap fail + the ~P10 relative-metric floor, record the threshold where
  the metrics core or settings exposes it. Blocked by this ticket.
- **BPIR-refinement** and **edge-crossing-reduction** follow-ups → filed by
  the calibration ticket once it has measured the gap (they are seeded with
  that data, so they cannot be filed here).

## Severity justification

**High.** Reach: the util underpins **every** MCP-generated graph
(every-session path), which bumps impact up. Impact class: hard blocker with
no workaround — there is no `FGraphLayoutMetrics` substitute anywhere in
source and seven dependent feature tickets gate on it (verified
`blockedBy: [F-graph-layout-metrics-core]` on each). Not Critical because it
neither crashes the editor nor corrupts asset data; not Medium because the
gap blocks a whole dependent cluster with no documented workaround.

## History
- `#1-initial-spec` `OPEN` reporter — No objective layout-quality metric exists; B-node-layout-poor closed on a trivial 4-node check while real graphs still lay out badly. Build a generalized node-size estimator (from FNodeLayoutEngine::EstimateNodeSize) + a shared FGraphLayoutMetrics util, calibrate thresholds against human-authored vs MCP corpora, then file BPIR-refinement and edge-crossing follow-ups.
- `#2-rescope` `OPEN` developer — Rescoped (was over-scoped: bundled an open-ended threshold-calibration pass + two measurement-seeded follow-up-ticket filings into one unit, neither of which any of the 7 dependents block on). Now scoped to the two code deliverables the dependents actually consume — the pure engine-agnostic `FGraphLayoutMetrics` util and the node-size adapter seam (interface + Blueprint adapter wrapping the existing `EstimateNodeSize`) — plus the clean-high/overlap-low unit test. Calibration split to new deferred `F-graph-layout-calibration`; the BPIR-refinement / edge-crossing follow-ups are filed by that ticket once it has measured the gap. Title/tags/severity-justification updated to match.
- `#3-implemented` `IN-REVIEW` developer — Implemented the rescoped util + seam. New `Private/Layout/GraphLayoutMetrics.{h,cpp}`: pure engine-agnostic `GraphLayout::ComputeGraphLayoutMetrics(Nodes[{nodeId,x,y,w,h}], Edges[{from,to}], GridSizePx)` returning overlap / spacing / straightness / edge-crossings / grid sub-scores + a weighted 0-1 combined score; overlap is the hard fail (any positive-area bbox intersection → overlap sub-score 0 + a non-empty list of overlapping node-id pairs); straightness/crossings are approximated from node centers (limitation documented in the header). Also adds the engine-agnostic `INodeSizeAdapter` seam and `Private/Layout/BlueprintNodeSizeAdapter.{h,cpp}`, the reference adapter wrapping the existing `BpirLayout::EstimateNodeSize` (no rewrite). Regression test `Private/Tests/Bpir/TestGraphLayoutMetrics.cpp` (3 cases: PinWright.layout.metrics.CleanVsOverlap asserts overlap>0 / sub-score 0 / strictly-lower combined on the bad graph vs a clean one; .EdgeCrossings asserts crossing diagonals are detected and penalized vs parallel edges; .BlueprintNodeSizeAdapter asserts the adapter returns the identical size to the direct estimator and tolerates null). Material/anim/rig adapter bodies remain with their bounds-aware tickets; calibration is the deferred `F-graph-layout-calibration`.
- `#4-parallel-host-reconcile` `IN-REVIEW` developer — A second fuzz host independently implemented the same ticket against a narrower `PinWright::GraphLayoutMetrics::Evaluate(FGraphLayoutNode/FGraphLayoutEdge)` API that omitted the node-size adapter seam. On rebase the two implementations collided in `GraphLayoutMetrics.{h,cpp}`; reconciled by keeping the `#3-implemented` superset (the `GraphLayout::ComputeGraphLayoutMetrics` API plus the `INodeSizeAdapter` seam and `BlueprintNodeSizeAdapter.{h,cpp}`, which satisfies more of the spec) as canonical and porting the parallel host's clean-vs-overlap regression test to that API as additional coverage at `Private/Tests/Layout/TestGraphLayoutMetrics.cpp` (alongside the existing `Private/Tests/Bpir/TestGraphLayoutMetrics.cpp`). No production behavior dropped from either side.
