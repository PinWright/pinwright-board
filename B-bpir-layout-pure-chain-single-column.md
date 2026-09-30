---
id: B-bpir-layout-pure-chain-single-column
title: "BPIR layout stacks every upstream pure node of a consumer in one column, top-aligned — a pure node feeding another pure node gets a wire that runs backwards, and no data wire is pin-aligned"
status: OPEN
severity: Medium
category: bug
tags: [layout, bpir, blueprint, pure-nodes, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# BPIR layout puts chained pure nodes in one column (backward wires)

For each control-flow consumer, BPIR's post-compile layout collects every pure
(data-only) node upstream of it, breadth-first. It then places all of them in a
**single column**: right-aligned to the consumer's left edge minus a small gap,
stacked downwards in collection order
(`Source/PinWright/Private/Compiler/NodeLayoutParameterFormatter.cpp:128-157`).

This causes three visible problems:

1. **Backward wires on multi-level chains.** Take `GetActorLocation → VectorLength
   → Greater → Branch.Condition`. All three pure nodes sit in the same column, so
   the wire from `GetActorLocation` (lower in the column) to `VectorLength`'s input
   (same X) loops out of the right edge and back into the left edge.
2. **No pin alignment.** The first pure node takes the consumer's top Y
   (`:153-155`), and the rest stack below it. No data wire is made horizontal
   against the pin it feeds.
3. **Order does not follow the consumer's pins beyond the first level.** Deeper
   ancestors interleave in breadth-first order, which adds crossings.

Found by reading the code; not yet reproduced live.

**Fix:** data nodes cascade leftwards one column per dependency level, and each is
aligned to the row of the pin it feeds where possible (`F-graph-layout-core`,
algorithm steps 4 and 6).

**Acceptance:** on a 3-deep pure-chain fixture:
- 0 backward data edges;
- every first-level data edge has |source pin Y − target pin Y| ≤ 1 px when
  measured pin offsets are available (`F-graph-node-size-measured`);
- 0 overlaps.

## Severity justification

**Medium.** This fires on every BPIR body with nested pure expressions, which is
the common case. The damage is readability, not data.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30 (code reading): all upstream pure nodes of a consumer go into one right-aligned column, top-aligned to the consumer (NodeLayoutParameterFormatter.cpp:128-157), so pure→pure wires run backwards and no data wire is pin-aligned. Fix via F-graph-layout-core.
