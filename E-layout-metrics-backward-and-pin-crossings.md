---
id: E-layout-metrics-backward-and-pin-crossings
title: "GraphLayoutMetrics cannot see the most visible layout defects: no backward-edge count, and crossings/straightness are computed from node centres instead of pin rows"
status: OPEN
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
