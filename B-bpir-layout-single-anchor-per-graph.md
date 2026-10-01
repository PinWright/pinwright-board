---
id: B-bpir-layout-single-anchor-per-graph
title: "BPIR's post-compile layout formats only the first event chain in each graph — every other entry compiled into the same event graph keeps raw emitter placement and can be overlapped by the formatted chain"
status: IN-REVIEW
severity: Medium
category: bug
tags: [layout, bpir, blueprint, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# BPIR layout formats only one event chain per graph

`RunLayoutPass` (`Source/PinWright/Private/Compiler/BpirCompiler.cpp:1892-2037`)
runs once per graph per compile. It chooses **one** anchor: the first
event/entry/tunnel node that the compile created, in `Graph->Nodes` order
(`:1947-1956`). The layout then walks only the control-flow links reachable from
that anchor.

When one `compile_bpir` call creates several entries in the same event graph (for
example `ReceiveBeginPlay`, `ReceiveTick` and a custom event):

- **The first chain is formatted; the rest are not.** The other chains are in the
  layout pool but are never reached. They keep the emitter's raw placement:
  - event at x=0, y = the lowest node top so far + 450;
  - +450 px per control-flow node;
  - data nodes at −300, stacked 130 px apart whatever their size
    (`CodeNodeEmitter.cpp:117-128, 162-178`).
- **The formatted chain can overlap them.** Its collision checks consider only
  nodes it has already placed plus pre-existing nodes. The unreached pool nodes are
  neither, so they are not obstacles. When the formatted chain's branches stack
  downwards they can land on the next event's chain.

Found by reading the code; not yet reproduced live.

**Repro (expected):** compile one BPIR text with three entry blocks, each with a
Branch and some data inputs, into an empty event graph. Only the first block is
laid out; the other two keep the 450 px stride; overlaps are likely.

**Fix:** lay out every root in the created pool — each event/entry/tunnel, plus
the upstream root found by the insert-mode walk — and stack the trees without
overlap. `F-graph-layout-core` does this by construction. A stop-gap is to loop
the existing pass per root, feeding the rects of already-formatted trees in as
obstacles.

**Acceptance:** a 3-entry fixture compiled in one call gives 0 overlaps in
`GraphLayoutMetrics`, and all three chains are formatted (straight main rows),
not just the first.

## Severity justification

**Medium.** Multi-entry compiles are common, and the result is a visibly broken
layout with no error. Nothing is lost or corrupted.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30 (code reading): RunLayoutPass picks a single anchor per graph (BpirCompiler.cpp:1947-1956) and the layout walks only exec links from it, so other entry chains created by the same compile keep CodeNodeEmitter's raw 450 px stride and are not collision obstacles for the formatted chain. Fix via F-graph-layout-core (per-root trees).
- `#2-fixed-by-layout-core` `IN-REVIEW` developer — Fixed by F-graph-layout-core: `RunLayoutPass` (`Compiler/BpirCompiler.cpp`) passes every created event / function entry / entry tunnel as a root (insertion or body anchor first, the rest by position), and `PwGraphLayout` lays out one tree per root, stacking later trees below earlier ones and around every pre-existing node. Tests: `PinWright.layout.blueprint.CompilePassArrangesEveryEntry`, `PinWright.layout.core.ThreeEvents`.
