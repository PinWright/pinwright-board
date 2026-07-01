---
id: E-get-node-details-batch-undiscovered-on-read-path
title: "get_node_details_batch is framed only as an after-creation tool, so reverse-engineering tasks inspect N known nodes with N singular get_node_details calls"
status: WONTFIX
severity: Low
category: ergonomic
tags: [blueprint-graph, get_node_details, get_node_details_batch, batch, docs, reverse-engineering, excessive-steps]
---

# `get_node_details_batch` is undiscovered on the read/reverse-engineering path — agents call singular `get_node_details` once per node

`blueprint.graph.get_node_details_batch` already exists (`nodeIds: ["<id>", ...]`
→ details for every node in one call). But every place it is surfaced frames it
as a **post-creation** tool — the read-the-pin-shape-of-the-node-I-just-made
case:

- `docs/wiki-src/blueprint.graph.md` line 7 (the only mention): *"After a
  creation, `blueprint.graph.get_node_details` (or `_batch`) returns the actual
  pin shape so the next `connect_pins` call doesn't fail."*
- `E-graph-standard-exec-pin-names` and `E-pin-details-batch-requests-shape-mismatch`
  likewise reference `_batch` only in the authoring/connect flow.

There is **no H3 overlay section** for `get_node_details`, `get_nodes`, or
`get_node_details_batch` in `blueprint.graph.md` — so an agent that navigates to
the `get_node_details` method page while *reading* an existing graph (the
reverse-engineering / "document how this BP works" intent) gets no steer toward
batching a set of already-known node ids. The natural call it lands on is the
singular `get_node_details`, once per node of interest.

The symmetric read-path case — "I already have a list of N node ids and want all
their details" — is exactly what `get_node_details_batch` is for, but it is
invisible from the singular method's own page and from the namespace prelude's
read-oriented guidance.

## What it should do

Lightweight docs/ergonomic fix (no code behavior change — the batch method works):

1. In `docs/wiki-src/blueprint.graph.md`, add a short `### blueprint.graph.get_node_details`
   H3 overlay section whose body cross-refs the batch sibling for the multi-node
   case: *"Inspecting several known nodes? Pass all their ids to
   `get_node_details_batch` (`nodeIds: [...]`) in one call instead of calling this
   per node."* (H3 sections cost no tokens on the namespace page — they surface
   only when an agent calls `call("blueprint.graph.get_node_details")` directly,
   which is exactly the discovery moment.)
2. Optionally broaden the line-7 framing from "After a creation" to also cover
   the read/reverse-engineering path, so the batch sibling is presented as the
   default for *any* multi-node detail dump, not only post-creation pin checks.

## Evidence

mcp-fuzz process audit of a `blueprint.graph` reverse-engineering task on
`BP_Timeline_Ball` (16 calls, outcome clean, self-reported friction "none").
The agent inspected four key nodes — the Timeline node, a SetRelativeLocation
node, a SetRelativeScale3D node, and a SpawnEmitter node — with **four
sequential singular `get_node_details` calls**:

- `blueprint.graph.get_node_details` — *"Timeline node AFB695..."*
- `blueprint.graph.get_node_details` — *"SetRelativeLocation node 62B320..."*
- `blueprint.graph.get_node_details` — *"SetRelativeScale3D node 59FADB..."*
- `blueprint.graph.get_node_details` — *"SpawnEmitter node 56A0C3..."*

All four node ids were already known (the agent had just paged the full
`get_nodes` list to a file). A single `get_node_details_batch(nodeIds=[AFB695,
62B320, 59FADB, 56A0C3])` would have replaced the four round-trips with one.
This is the taxonomy's "N calls where a single batch should exist" — except the
batch exists; it just isn't discoverable from the read path. No error, no
retry, no misuse-then-correct, so it never surfaced as friction in the report —
which is exactly why it slips through: a clean-but-3-round-trips-too-long read
pattern that repeats on every multi-node inspection / reverse-engineering task.

Distinct from `E-pin-details-batch-requests-shape-mismatch` (that ticket is the
param-shape divergence between the two *batch* methods causing a misuse-then-
correct; this is the singular-method-used-N-times discoverability gap on the
read path) and from `E-graph-standard-exec-pin-names` (exec-pin-name guidance in
the authoring flow).

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of the `blueprint.graph` BP_Timeline_Ball reverse-engineering task (16 calls, outcome clean, friction self-reported "none"): agent inspected 4 already-known node ids (Timeline / SetRelativeLocation / SetRelativeScale3D / SpawnEmitter) with 4 sequential singular `get_node_details` calls where one `get_node_details_batch(nodeIds=[...])` would do. Root cause is discoverability: `get_node_details_batch` exists but is framed only as a post-creation pin-check tool (wiki line 7 "After a creation…"), and `blueprint.graph.md` has no H3 overlay section on `get_node_details` to steer the read/reverse-engineering path toward the batch sibling. Propose: add a `### blueprint.graph.get_node_details` overlay H3 cross-reffing `get_node_details_batch` for the multi-node case, and broaden the line-7 framing beyond "After a creation". Docs/ergonomic only — the batch method works. Distinct from E-pin-details-batch-requests-shape-mismatch (param-shape divergence / misuse-then-correct) and E-graph-standard-exec-pin-names (authoring-flow exec-pin naming).
- `#2-wontfix` `WONTFIX` developer — Won't fix: the adversarial "worth it" gate is decisive. The batch method already exists and works (`BlueprintGraphInspectionHandler.cpp:651-718`); the singular it would replace is at `:440`. The sole evidence is ONE reverse-engineering task with outcome **clean** and self-reported friction **"none"** — the ticket body concedes "No error, no retry, no misuse-then-correct, so it never surfaced as friction." This is friction inferred by auditing an otherwise-successful task (3 redundant read round-trips, zero agent-perceived pain), the textbook cosmetic discoverability nit. The source claims all check out (line-7 "After a creation…" framing at `wiki-src/blueprint.graph.md:7`; no `### blueprint.graph.get_node_details` H3 — only `### blueprint.graph.get_execution_flow` at :343), but actioning it still costs an overlay-edit + regression-test churn cycle plus the fragile-parse placement care: the overlay prelude boundary is the FIRST `### ` line (`WikiOverlay.cpp:111-116`), which in this file is `### Params` at :56 — a new H3 must be appended at EOF to parse cleanly. Not worth the churn against zero observed friction. The only sliver of merit (a one-line broadening of the line-7 framing) is too small to justify a standalone ticket. Distinct, non-duplicate, no regression — declined on the worth-it lens, not on novelty.
