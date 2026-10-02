---
id: E-layout-metrics-backward-and-pin-crossings
title: "GraphLayoutMetrics cannot see the most visible layout defects: no backward-edge count, and crossings/straightness are computed from node centres instead of pin rows"
status: DONE
severity: Medium
category: ergonomic
tags: [layout, metrics, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Layout metrics: add backward edges and pin-row crossings

`GraphLayout::ComputeGraphLayoutMetrics`
(`Source/PinWright/Private/Layout/GraphLayoutMetrics.{h,cpp}`) scores overlap,
spacing, straightness, crossings and grid regularity. Its own header says
straightness and crossings are approximated from node *centres*
(`GraphLayoutMetrics.h:9-12`).

Two gaps:

1. **No backward-edge measure.** A wire whose target is upstream of its source, in
   the graph's flow direction, is the defect a reader notices first. It is the
   dominant defect in `B-mgir-layout-grows-away-from-root` and
   `B-bpir-layout-pure-chain-single-column`, and the metric has no term for it.
2. **Centre-based geometry.** Two nodes can have aligned centres while their
   connected pins are rows apart, and crossings between centre-to-centre lines
   differ from crossings between pin-to-pin wires. A pin-aligned layout
   (`F-graph-layout-core`) cannot be verified by a centre-based metric. The layout
   and the check would agree with each other and disagree with the picture
   (rpc-design §4).

**Fix:**
- Accept optional pin anchors per edge: `{fromNode, fromPinY, toNode, toPinY}`
  plus the flow direction.
- Add `backwardEdges` (a count plus the offending edge list) and `pinRowDeltaPx`
  per edge.
- Compute crossings on pin-to-pin segments when anchors are present, falling back
  to centres otherwise, and report which basis was used.

**Acceptance:** unit tests in both failure directions:
- a fixture with one reversed edge reports `backwardEdges = 1`, and the corrected
  fixture reports 0;
- two layouts with identical centres but different pin alignment score
  differently;
- the pin-based crossing count differs from the centre-based count on a
  constructed case.

## Severity justification

**Medium.** Without these terms, `layout_report` and the auto-layout verbs'
before/after metrics cannot confirm the fixes they exist to check.

## History
- `#1-initial-spec` `OPEN` reporter — Gap analysis 2026-09-30: GraphLayoutMetrics has no backward-edge term and computes straightness/crossings from node centres (GraphLayoutMetrics.h:9-12); add optional pin anchors, backwardEdges, per-edge pin-row delta, pin-to-pin crossings with the basis reported.
- `#2-backward-and-pin-geometry` `IN-REVIEW` developer — `Layout/GraphLayoutMetrics.{h,cpp}`: `FGraphEdge` gains optional pin anchors (`FGraphEdge(from, fromPinY, to, toPinY)`, pin centre from each node's top) and `ComputeGraphLayoutMetrics` an `EFlowDirection` (default LeftToRight). Anchored edges are drawn output side -> input side at the pin rows; straightness and crossings use those segments, unanchored edges fall back to centres, and `GeometryBasis` reports `pins` / `centers` / `mixed`. New result fields: `BackwardEdgeCount` + `BackwardEdges` (target's input side upstream of the source's output side, self-edges excluded) and `PinRowDeltaPx` (parallel to the edges, -1 for unknown ids). Existing callers unchanged (new param defaulted). Tests (new file `Tests/Layout/TestGraphLayoutMetricsPins.cpp`): `PinWright.layout.metrics.BackwardEdgeCounted` (reversed fixture 1 / corrected 0 / mirrored direction 2), `PinWright.layout.metrics.PinAlignmentChangesStraightness` (identical rects, different pin rows: straightness and combined score differ, row delta 0 vs 64, centre basis cannot tell them apart), `PinWright.layout.metrics.PinCrossingsDifferFromCentres` (swapped output rows cross once pin-to-pin, 0 by centres; mixed basis). Offline harness: all pass; ignoring anchors and dropping the backward list turns all three red. No `layout_report` verb exists in the tree, so nothing publishes these yet beyond tests and the core's callers. Every changed/new TU compile-checked with UBT -SingleFile (Linux, UE 5.8): all succeed. Not yet run in an editor (manager owns the build/test slot).
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). All three Acceptance tests passed in w23-final: `PinWright.layout.metrics.BackwardEdgeCounted` (reversed fixture 1, corrected 0, mirrored direction 2), `PinWright.layout.metrics.PinAlignmentChangesStraightness` (identical centres, different pin rows score differently; row delta 0 vs 64) and `PinWright.layout.metrics.PinCrossingsDifferFromCentres` (one pin-to-pin crossing, zero by centres, `mixed` basis). The five older metrics tests also passed. Note: no `layout_report` verb publishes these fields yet; only the layout core and tests consume them.
