---
id: F-controlrig-layout-bounds-aware
title: "Bounds-aware ControlRig-graph layout (FCRIRLayoutEngine)"
status: OPEN
severity: Medium
category: feature
tags: [layout, controlrig, crir, rigvm, node-size]
blockedBy: [F-graph-layout-metrics-core]
encounters: 1
lastSeen: 2026-06-24T19:46:41Z
---

# Bounds-aware ControlRig-graph layout (FCRIRLayoutEngine)

`FCRIRLayoutEngine` lays out RigVM nodes without node-size awareness:
`Source/PinWright/Private/CRIR/CRIRLayoutEngine.cpp` (`FCRIRLayoutEngine::RunLayout`
from line 67; spec-cited body ~75-150). The engine already has
**pre-positioned-obstacle awareness** — depth computation walks the whole
graph including pre-positioned obstacles (comment at lines 86-87), and lanes
track pre-positioned siblings (lines 104-105) — so column boundaries are
defined by obstacles while nodes-to-layout are repositioned. That obstacle
awareness must be **kept**; this ticket only adds real node dimensions on top
of it. Without measured bounds, wide/tall rig nodes overlap.

## What it should do

- Place RigVM nodes using real measured bounds (column widths / row heights)
  from the metrics-core node-size estimator's rig-node adapter
  (`F-graph-layout-metrics-core`), replacing fixed-grid stepping.
- **Preserve** the existing pre-positioned-obstacle handling: obstacles still
  define column boundaries; only nodes-to-layout move, now sized by measured
  bounds rather than constants.
- Finish with a shared overlap-resolution sweep (the `FormatY`-style
  collision approach, `FNodeLayoutEngine::FormatY`,
  `Source/PinWright/Private/Compiler/NodeLayoutEngine.cpp:1268`).

## Proposed approach

- Thread the node-size estimator through `FCRIRLayoutEngine::RunLayout`,
  using measured widths/heights for both column placement and obstacle-gap
  computation so sized nodes do not collide with pre-positioned obstacles.
- Reuse the shared collision-resolution sweep; do not duplicate it.
- **Regression test:** run auto-layout on a real ControlRig graph that
  contains pre-positioned obstacle nodes, then call `FGraphLayoutMetrics` and
  assert `overlap == 0` (including against the obstacles) and an improved
  combined score versus the baseline.

## Severity justification

**Medium.** Soft blocker: ControlRig auto-layout works but overlaps on
non-trivial graphs; cleanup is manual. No crash, no data corruption. Blocked
by `F-graph-layout-metrics-core` for the estimator and the metrics assertion.

## History
- `#1-initial-spec` `OPEN` reporter — FCRIRLayoutEngine (CRIRLayoutEngine.cpp:75-150) has no node-size awareness but already tracks pre-positioned obstacles; add bounds-aware placement (metrics-core estimator) while keeping obstacle awareness + a FormatY-style overlap sweep, verified by FGraphLayoutMetrics asserting no overlap.
