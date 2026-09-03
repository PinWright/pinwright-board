---
id: E-node-connections-rejects-graphname
title: "blueprint.get_node_connections rejects graphName with [UNKNOWN_PARAMS] while its read-path sibling blueprint.graph.get_node_details requires it — the same node id is addressed two incompatible ways one call apart"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, get_node_connections, get_node_details, graphname, unknown-params, param-asymmetry, read-path, discoverability]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Two node-inspection verbs, one node id, opposite rules about `graphName`

Inspecting a single node on the blueprint read path routinely takes both verbs back to back —
`blueprint.get_node_connections` to see what a pin is wired to, `blueprint.graph.get_node_details`
to see what the node *is*. They disagree about how to address the node:

- `blueprint.graph.get_node_details` **requires** `graphName`.
- `blueprint.get_node_connections` **refuses** it: passing the same `graphName` the previous call
  demanded returns `[UNKNOWN_PARAMS]`.

So the caller writes the pair, has the second call rejected by the strict parameter gate, deletes
one key, and retries. Every time. The rejection is at least loud and the fix is obvious from the
error's own valid-parameter list — this is friction, not a trap — but it is friction on two verbs
that are used together by construction.

## Where it was met

During a WEAPONS critic review of `/Game/FPS/Weapons/BP_WeaponBase`, disambiguating which pin of a
`Break Hit Result` node fed a downstream comparison (see `B-bpir-break-struct-pin-not-named`, which
this round-trip was serving). The node id was already in hand; both verbs were called against it in
sequence, and the `graphName` that the details call needed was the key the connections call threw
out.

## What is asked for

Accept `graphName` on `blueprint.get_node_connections` as an **optional disambiguator** — used when
supplied, ignored when the node id resolves uniquely without it — so a caller can pass one argument
set to both verbs. That is the cheaper and safer half of the fix: it changes no resolution semantics
for anyone who omits it, and it does not require the two verbs to agree on whether `graphName` is
mandatory.

If accepting the key is judged wrong (e.g. because node ids are already globally unique on this
path and the parameter would be dead weight), then the ask is documentation instead: say so on the
`blueprint.get_node_connections` page, explicitly, next to the `nodeId` parameter — "no `graphName`;
node ids resolve blueprint-wide here, unlike `blueprint.graph.*`". Silence is what makes the caller
try it.

## Root cause — GUESS, not source-read

No source was read. The likely mechanism is simply that `get_node_connections` lives in the
top-level `blueprint` namespace and never declared `graphName` in its `RPC_PARAMS` block, so the
dispatcher's strict unknown-parameter gate rejects it — the same shape as
`B-orbit-shots-no-subject-coverage`, where a parameter a sibling verb declares is refused because
this one's block does not list it.

**Workaround:** drop `graphName` from the `get_node_connections` call; keep it on the
`blueprint.graph.get_node_details` call. Two argument shapes for one node.

## Related

- `E-blueprint-node-verb-alias-param-drift` (OPEN, Low) — the **authoring** half of the same
  `blueprint.*` vs `blueprint.graph.*` split: `add_node`/`connect_pins` vs
  `graph.create_node`/`graph.connect_pins`, disagreeing on every parameter name with no overlay
  reconciling them. This ticket is that same namespace split showing up on the **read** path, and
  the two together argue the cross-namespace parameter vocabulary should be reconciled once rather
  than verb by verb.
- `E-graphname-rejects-event-name` (IN-REVIEW) — the *other* `graphName` friction: an accepted
  `graphName` that is an **event** name gets `GRAPH_NOT_FOUND`. Distinct — there the parameter is
  accepted and the value is wrong; here the parameter itself is refused.
- `B-orbit-shots-no-subject-coverage` (OPEN) — the same "sibling verb declares it, this one does
  not, strict gate refuses it" shape on the render side.
- `E-blueprint-param-name-path-vs-assetpath` (OPEN) — adjacent naming drift on the same namespace.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Met during a WEAPONS critic review of `/Game/FPS/Weapons/BP_WeaponBase`, while using both node-inspection verbs back to back on one node id to disambiguate a `Break Hit Result` pin (the round-trip filed as `B-bpir-break-struct-pin-not-named`). `blueprint.graph.get_node_details` **requires** `graphName`; `blueprint.get_node_connections` **refuses** the same key with `[UNKNOWN_PARAMS]`. So the same node is addressed two incompatible ways one call apart, and the caller must strip a key between two calls that are used together by construction. Rejection is loud and the error lists valid parameters, so this is friction rather than a trap — hence Low. Root cause is an explicit GUESS, no source read: `get_node_connections` lives in the top-level `blueprint` namespace and presumably never declared `graphName` in its `RPC_PARAMS` block, so the strict unknown-parameter gate refuses it — the same shape as `B-orbit-shots-no-subject-coverage`. Ask: accept `graphName` as an optional disambiguator (used when supplied, ignored when the node id resolves uniquely), which changes no semantics for callers who omit it; or, if the parameter is genuinely meaningless on this path, say so explicitly on the `blueprint.get_node_connections` page beside `nodeId`, because silence is what makes callers try it. Read-path counterpart to `E-blueprint-node-verb-alias-param-drift`, which records the same `blueprint.*` / `blueprint.graph.*` vocabulary split on the authoring path.
