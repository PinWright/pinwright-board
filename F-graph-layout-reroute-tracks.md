---
id: F-graph-layout-reroute-tracks
title: "Optional reroute (knot) insertion for long, backward or node-crossing wires after graph layout"
status: OPEN
severity: Low
category: feature
tags: [layout, reroute, knot, blueprint, deferred, gap-analysis-2026-09-30]
blockedBy: [F-graph-layout-core]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Optional reroute insertion after graph layout

**Deferred.** Gated on `F-graph-layout-core`. Pick this up only after that
layout's backward-edge and overlap metrics are clean on the fixture suite. Most
long or backward wires should disappear with a correct layered layout, so measure
what remains before building this.

**The problem.** Wires that stay long, backward (cycle back edges, loops) or pass
behind unrelated nodes after layout are hard to follow. `blueprint.graph.create_reroute_node`
exists, but nothing places reroutes automatically.

**What it should do** (opt-in, off by default):

1. After layout, find wires that are longer than a threshold, run backwards, or
   whose segment intersects a node rect.
2. Route each one along a horizontal track: at a source or target pin row when that
   row is clear, otherwise at the first clear Y below the obstacles.
3. Stack parallel tracks a fixed spacing apart, and insert reroute nodes at the
   track's ends.
4. Be deterministic.
5. Report `reroutesCreated[]`, and refuse on graph types without a reroute node.

**Acceptance:** on a fixture with an exec loop and a wire passing behind a node,
enabling the option:
- removes every node-crossing segment;
- the loop's back wire routes above or below the loop body;
- two runs create identical reroutes;
- undo removes them.

## Severity justification

**Low.** Readability polish that depends on the core layout landing first.

## History
- `#1-initial-spec` `OPEN` reporter — Gap analysis 2026-09-30: deferred (blockedBy F-graph-layout-core) opt-in reroute-track insertion for long/backward/node-crossing wires; build only if residual defects remain after the layered layout lands.
