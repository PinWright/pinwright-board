---
id: F-graph-batch-delete-clear-mode
title: "No batch node delete / clear-graph authoring mode forces per-node stub cleanup before a clean BPIR round-trip"
status: OPEN
severity: Low
category: feature
tags: [blueprint, graph, delete_node, compile_bpir, roundtrip]
encounters: 1
lastSeen: 2026-06-29T01:08:25Z
---

# No batch node delete / clear-graph authoring mode forces per-node stub cleanup before a clean BPIR round-trip

Every fresh Actor Blueprint ships 3 default disabled event stubs
(`ReceiveBeginPlay` / `ReceiveActorBeginOverlap` / `ReceiveTick`) that pollute any
decompile-based round-trip comparison, but the only way to remove them is **one
`blueprint.graph.delete_node` call per nodeId** — its schema takes a single
`nodeId`, no array — and `blueprint.compile_bpir` `mode=append` only deletes
entries with a *matching signature*, so it cannot clear these unrelated default
stubs. So a clean-slate authoring start (the canonical first step for the
round-trip-fidelity probes this suite runs repeatedly) costs a fixed
~6-call detour: `decompile` → `get_nodes(namesOnly)` → 3× `delete_node` →
verify `decompile`, all before the first authoring call.

**What it should do (either):**
- Add a batch form to `blueprint.graph.delete_node` — accept a `nodeIds` array
  (and/or a `nodeTitles` / `disabledStubsOnly` selector) so the three stubs go in
  one call; or
- Add a `compile_bpir` option to start from a **blank / cleared** EventGraph
  (e.g. `mode: "replace_graph"` or a `clearGraph: true` flag) so a clean-slate
  round-trip is a single authoring call with no pre-cleanup.

**Workaround:** call `blueprint.graph.delete_node` once per stub nodeId after a
`get_nodes(namesOnly)` lookup.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a `blueprint.compile_bpir` authored-position round-trip-equivalence probe (`/Game/BP_RoundTripProbe`, fresh Actor BP, 24 nodes). CallAnalyzer of the Attempt transcript (`agent-ad1dca8b223361239.jsonl`, 24 `mcp__pinwright__call`) shows the agent spent ~6 calls clearing the 3 default disabled stubs before authoring: `decompile` (saw 3 ghost events) → `get_nodes(namesOnly)` → 3 consecutive `delete_node` calls (nodeIds `BDFD81C3…`, `7040AC2C…`, `BBEDD906…`, each returning `newOrphanedCount:0`) → verify `decompile`. `compile_bpir mode=append` cannot remove the stubs (no matching signature) and `delete_node` exposes only a single `nodeId` (no array), forcing one call per node. Distinct from `E-inspect-events-omits-disabled-stub-flag` (that ticket is about the readback not *flagging* the stubs; this is about the write side having no batch/clear verb to *remove* them efficiently). Friction note verbatim: *"~6 calls of overhead before any authoring."* Severity Low: pure convenience batch with a cheap per-node workaround, but it is a recurring setup tax for this suite's round-trip probes.
