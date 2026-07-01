---
id: F-bp-graph-integrity-snapshot
title: "No bundled graph-integrity snapshot — decompile + connections + orphan scan must be called as a 3-call trio at every checkpoint"
status: OPEN
severity: Low
category: feature
tags: [blueprint-graph, ergonomics, batching, verification, integrity]
encounters: 3
lastSeen: 2026-06-28T23:48:41Z
---

# No single graph-integrity snapshot call

Verifying graph integrity — a recurring need before/after any graph mutation —
requires calling three separate read RPCs as a fixed trio:
`blueprint.decompile` + `blueprint.graph.get_graph_connections` +
`blueprint.graph.find_orphaned_nodes`. An Attempt that checkpoints integrity at
several points pays 3 calls per checkpoint, and needs all three together to
cross-check topology (e.g. to distinguish a decompiler display glitch from real
corruption — only the trio agreeing is conclusive).

**Proposed:** a convenience RPC `blueprint.graph.snapshot` (or
`blueprint.graph.integrity_report`) returning the decompiled BPIR + connection
edges + orphan list in one response, collapsing each checkpoint from 3 calls to
1. Pure additive batching over the three existing read paths; no new analysis.

## Evidence

From this task (`blueprint.compile_bpir` failure-rollback probe, transcript
`agent-ac9ff02aeac548e24.jsonl`): the agent ran the identical trio at three
checkpoints — baseline, after malformed re-upsert #1, after malformed re-upsert
#2 (calls 6-8, 10-12, 14-16 = 9 calls). THINK noted it needed "three
independent authoritative reads agree" to confirm corruption vs. a display
glitch. No single call returns decompiled BPIR + edges + orphan scan together.

## Severity

**Low.** Pure ergonomic batching, not a defect — each of the three calls
returns distinct, genuinely-needed data, and the trace was otherwise smooth
(no retries, no guessed params). The cost is round-trips, not correctness.
Reach is broad (graph-integrity verification recurs across most BPIR/graph
mutation tasks), but the impact per occurrence is just 2 extra calls per
checkpoint, so it stays Low.

## History
- `#1-initial-feature-request` `OPEN` reporter — Filed from the CallAnalyzer trace of the `blueprint.compile_bpir` failure-rollback probe. Graph-integrity verification forced a fixed `blueprint.decompile` + `blueprint.graph.get_graph_connections` + `blueprint.graph.find_orphaned_nodes` trio at every checkpoint (run 3× here = 9 calls; trace calls 6-8/10-12/14-16). Agent THINK: needed all three views together ("three independent authoritative reads agree") to distinguish a decompiler display glitch from real corruption. Proposed a `blueprint.graph.snapshot` convenience returning all three in one response. Ergonomic batching gap only — no retries, no defect.
- `#2-cross-task-evidence` `OPEN` reporter — Same gap recurs on a separate `blueprint.compile_bpir` idempotency probe (`/Game/BP_BpirIdempotent`, transcript `agent-a04ca8ba0fa018d4f.jsonl`, 14 RPCs, zero errors/retries — fully clean trace). Proving a byte-identical re-apply was a true no-op forced a fixed 3-read readback trio at each of two checkpoints (baseline + after re-apply) = 6 calls: `blueprint.graph.get_nodes` (count/positions/single-entry) + `blueprint.decompile` (body) + `blueprint.graph.find_orphaned_nodes` (no-orphan proof). Note the trio's third member VARIES by task (`get_nodes` here vs `get_graph_connections` in `#1`) while the underlying need — one graph-integrity/state snapshot per checkpoint — is constant; the CallAnalyzer independently sketched the same convenience ("a single graph-state digest/diff returning {nodeCount, entryPoints, orphanCount, position-hash, decompiled-body}"). Suggests the proposed `blueprint.graph.snapshot` should also surface node count + positions + entry-point list, not just decompile+connections+orphans. Reach corroboration on a smooth trace; no friction, no retries — stays Low.
- `#3-idempotency-digest-corroboration` `OPEN` reporter — Same gap recurs on another clean `blueprint.compile_bpir` re-apply idempotency probe (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPCs all `ok`/non-error, zero retries; transcript `agent-a6a469a604352e7e0.jsonl`) on a fresh Actor BP `/Game/BP_BpirIdempotencyStress` (3 events with pure data-pin helpers → a 28-node EventGraph). To prove three identical upsert applies were a true no-op the task ran a per-apply readback — `blueprint.graph.get_nodes` + `blueprint.graph.find_orphaned_nodes` ×3 (plus one final `blueprint.decompile`) — then had to cross-diff node multisets+positions across applies by hand: each `get_nodes` overflowed the inline budget and spilled (`E-get-nodes-pins-spill-no-projection #11`), forcing a `Read` of each dump plus a `bash sort|diff`, and one pass produced a **false diff from a hand-typed baseline** before re-extracting from JSON. Exactly the case the `#2`-proposed graph-state digest (`{nodeCount, orphanCount, position-hash, entryPoints}`) would collapse: an idempotency comparison would be a single inline digest-vs-digest per apply instead of get_nodes-spill + manual sort/diff. Strengthens `#2`'s "surface node count + positions + a comparable fingerprint" angle on the byte-identical-re-apply scenario; the projection-lever fix lives on `E-get-nodes-pins-spill-no-projection`, the one-call digest/snapshot fix here. Reach corroboration on a smooth trace — stays Low.
