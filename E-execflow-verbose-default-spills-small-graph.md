---
id: E-execflow-verbose-default-spills-small-graph
title: "get_execution_flow with includeAllEntryPoints spills a tiny 9-node graph over the 10k display threshold, forcing a file-spill and a method switch"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, execution-flow, response-size, spill, verbosity, includeAllEntryPoints]
encounters: 1
lastSeen: 2026-06-29T04:26:55Z
---

# get_execution_flow's verbose default spills a small graph and pushes the caller to another method

`blueprint.graph.get_execution_flow` already supports `offset`/`pageSize` pagination
(it is cited as the *good* example in `E-get-nodes-pins-spill-no-projection`), yet with
`includeAllEntryPoints:true` its **default per-node representation is verbose enough that
even a tiny 9-node, 2-entry graph emits 15294 characters** and trips the 10000-char
`HttpResponses` spill (the mechanism added by DONE `E-http-response-spill`). The caller
gets back the small reference envelope `{outputTooLong:true, file.path:..., file.characters:15294,
file.threshold:10000}` instead of the flow inline — so on the natural reconvergence /
topology-verification path the result is unreadable without a `Read` of the spilled JSON,
and in practice the agent abandoned `get_execution_flow` entirely and switched to
`blueprint.graph.get_graph_connections (edgeType:exec)` to get the exec edges it needed.

Because pagination already exists, the gap is **not** "add pagination" — it is the verbose
default amplified by `includeAllEntryPoints` (which walks every entry's full chain and
concatenates them). A 9-node graph fitting inline should be the common case.

**Fix (pick one or more):** a more compact default node representation; a `summary`/
counts-only mode for `includeAllEntryPoints`; or auto-applying a sane `pageSize` so a small
graph's flow stays under the threshold. At minimum, document on `blueprint.graph.md` that
`includeAllEntryPoints` multiplies output (each entry's full chain) and that quick
reconvergence/topology checks may prefer `get_graph_connections (edgeType:exec)`.

## Evidence (from the audited round-trip-equivalence task on `blueprint.compile_bpir`)

Transcript `agent-a41684d4aee4673b2.jsonl` (focus `blueprint.compile_bpir`, namespace
`blueprint`). Call `get_execution_flow {graphName:"EventGraph", includeAllEntryPoints:true}`
on the 9-node, 2-entry-point `EventGraph` of fresh Actor BP `/Game/BP_BpirRoundTrip`
returned `{"outputTooLong":true, "message":"Response exceeds display limit (15294 chars,
threshold 10000); full payload written to ...HttpResponses/...json"}`. The agent did not
read the spilled file; it pivoted to `blueprint.graph.get_graph_connections (edgeType:exec)`
for the exec topology. Low: single occurrence, recovered immediately by switching methods —
but it is friction on the natural verification path for reconvergence checks, and
`get_execution_flow` is a common Blueprint-inspection verb.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `blueprint.compile_bpir` round-trip-equivalence task (transcript `agent-a41684d4aee4673b2.jsonl`). `get_execution_flow {graphName:"EventGraph", includeAllEntryPoints:true}` on a **9-node, 2-entry** graph (`/Game/BP_BpirRoundTrip`) emitted **15294 chars**, tripped the 10000-char `HttpResponses` spill (DONE `E-http-response-spill`), and returned only the `outputTooLong` reference envelope; the agent never read the spilled JSON and switched to `get_graph_connections (edgeType:exec)` for topology. `get_execution_flow` already has `offset`/`pageSize` (cited as the good example in `E-get-nodes-pins-spill-no-projection`), so the gap is the verbose default amplified by `includeAllEntryPoints` (walks every entry's full chain), not missing pagination. Fix: a more compact default / summary mode / auto-`pageSize` so a small graph fits inline; at minimum document the `includeAllEntryPoints` multiplier on `blueprint.graph.md`. Distinct from `E-execflow-no-event-entry-timeline-root` (timeline auto-start `NODE_NOT_FOUND`) and from the per-method spill family (`E-get-nodes-pins-spill-no-projection`, etc. — those lack pagination; this one has it and still spills). Severity Low (single occurrence, recovered immediately; pure friction on the verification path).
</content>
</invoke>
