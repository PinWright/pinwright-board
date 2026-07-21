---
id: F-pie-status-rpc
title: "No cheap PIE session status RPC: which PIE worlds exist and what map/state is each on"
status: OPEN
severity: Medium
category: feature
tags: [editor, pie, status, polling, multiplayer]
encounters: 1
lastSeen: 2026-07-21T00:00:00Z
---

# No cheap PIE session status RPC: which PIE worlds exist and what map/state is each on

No cheap read-only RPC answers "which PIE worlds exist and what map/state is
each on" — nothing enumerates PIE world contexts with per-context detail.
Origin: MP sumo netcode test sessions 2026-07-21, where agents needing to know
whether a servertravel/client-join had completed fell back to python
`ObjectIterator` scripts (same synchronous game-thread stall risk as the
console-command workaround in `F-console-command-world-target`) or blocking
python sleep loops.

**Workaround:** `python.execute` with `ObjectIterator`/world-context iteration
scripts; synchronous on the game thread, stalls the editor under polling.
**Fix:** Read-only `editor.pie_status` (final name may end up `pie.status` —
to be settled at implementation) returning one entry per PIE world context:
`{pieInstance, kind: server|client|standalone, netMode, mapName, worldPath,
gameStateClass, numPlayerControllers}`, empty when no PIE session is running.
Enables cheap client-side polling for "is the travel done" instead of blocking
python sleep loops. Note: implementation is already in flight by another agent
(filed alongside; developer flips to IN-REVIEW when done).

## History
- `#1-pie-status-polling-gap` `OPEN` reporter — Filed from MP sumo netcode test sessions: no RPC enumerates PIE contexts (instance, server/client kind, netMode, map, gameState, player-controller count), so travel-completion checks ran as game-thread-stalling python ObjectIterator scripts and blind sleep loops. Implementation is in flight by another agent; related `F-console-command-world-target`.
