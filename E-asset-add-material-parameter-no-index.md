---
id: E-asset-add-material-parameter-no-index
title: "asset.add_material_parameter returns parameterName but not the created expression's index/nodeId — forcing the caller to guess append order (0,1,2) for the index-addressed readback/wire, which breaks on a non-empty material"
status: OPEN
severity: Low
category: ergonomic
tags: [material, asset, add_material_parameter, node-index, no-echo, readback, response-shape, creation-verb-no-node-id]
encounters: 1
lastSeen: 2026-07-11T13:02:39.2618058+03:00
---

# `asset.add_material_parameter` omits the created expression's index — you must guess the append position before you can read or wire it

`asset.add_material_parameter` creates a real parameter expression
(`MaterialExpressionVectorParameter` / `ScalarParameter` / `StaticSwitchParameter`)
on a material graph, but its success response echoes only `{success, assetPath,
parameterName}` — it never returns the **index (or nodeId) of the expression it
just created**. Every downstream `asset.*` method that targets that expression is
**index-addressed**: `asset.get_material_node_details` takes `expressionIndex`,
and `asset.connect_material_pins` takes `fromExpression` (a number). So to read
the parameter back or wire it, the caller has to *guess* the index by assuming
the parameters were appended in call order (0, 1, 2).

That guess only holds because the material was **freshly created and empty**. On
any material that already has expressions, the append-order assumption is wrong
and the caller silently reads/wires the wrong node — with no id echoed at
creation to disambiguate.

## What it should do

Add the created expression's `index` (and/or a stable `nodeId`) to the
`asset.add_material_parameter` response — the way its `material.authoring` /
`material.graph` cousins echo `nodeId` on `create_node` / `add_*`. Then the
`add -> get_material_node_details -> connect_material_pins` sequence composes
deterministically on **any** material, not only a fresh empty one, with no
append-order guess. The handler already creates the expression locally; it just
lets the index go unreported.

## Why it's process friction (clean outcome, latent-wrong on a non-empty graph)

Surfaced in a clean `asset.add_material_parameter` task (focus
`asset.add_material_parameter`, namespace `asset`, outcome `tool_bug` — but the
judge's tool_bug is the neighbor `asset.get_material_node_details` stub, filed as
`B-material-get-node-details-missing-pins-props`; this index-omission is a
distinct process finding). Building `/Game/Materials/M_EnvProp_Master`, the three
`add_material_parameter` calls each returned only `parameterName` (Tint /
Roughness / UseDetail) with **no index**, so before calling
`get_material_node_details` the agent explicitly assumed the indices — SAY
verbatim: *"In a fresh empty material the added parameters should be indices 0, 1,
2."* — then read back nodes 0/1/2. The guess held only because the master was
empty; the CallAnalyzer flagged that on a non-empty graph the index-by-position
assumption would break.

Call cost this task: 3 `add_material_parameter` successes that each dropped the
index, forcing an index assumption before the 3 `get_material_node_details`
readbacks. No retries (the empty-graph guess happened to be right), so the cost
is latent risk + the reasoning detour rather than a wasted round-trip here.

severity rationale: impact=friction (creation verb omits the addressable
index/nodeId, forcing an append-order guess that is silently wrong on a non-empty
material) x reach=asset.* legacy material authoring, an off-canonical path -> Low.
Consistent with the sibling creation-verb-no-node-id ticket
`E-add-event-no-node-id-echo` (Low).

## Distinct from

- `B-material-get-node-details-missing-pins-props` (reopened this task) — that is
  the *readback* method `asset.get_material_node_details` returning a stub with no
  `parameterName`/defaults; this ticket is the *creation* verb omitting the index
  you need to call that readback at all. Different method, different root cause
  (missing return field on create vs stub payload on read).
- `E-add-event-no-node-id-echo` (OPEN) — same `creation-verb-no-node-id` family,
  but `blueprint.add_event` (GUID-addressed, different namespace/handler). This is
  the material-graph, index-addressed analogue with the extra append-order-guess
  wrinkle. Family sibling, not a duplicate.
- `E-material-mgir-bulk-path-undiscovered` (IN-REVIEW) — that ticket's remedy is
  "prefer the one-call MGIR bulk path over the asset.* imperative drip"; this is a
  specific response-shape gap in one asset.* verb, orthogonal to which path to
  take.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean `asset.add_material_parameter` task (focus `asset.add_material_parameter`, namespace `asset`, outcome tool_bug — judge filed the neighbor `B-material-get-node-details-missing-pins-props`, unrelated to this index-omission). Building master material `/Game/Materials/M_EnvProp_Master`, the 3 `add_material_parameter` calls each returned only `{success, assetPath, parameterName}` with NO created-expression index, while both downstream asset.* readback/wire methods are index-addressed (`get_material_node_details` takes `expressionIndex`, `connect_material_pins` takes `fromExpression`). The agent had to guess append order — SAY verbatim: "In a fresh empty material the added parameters should be indices 0, 1, 2." — before reading nodes 0/1/2; the guess held only because the master was freshly created and empty. From the CallAnalyzer's transcript analysis (efficiency finding 1, pattern "frustrating"): on a non-empty material graph the index-by-position assumption would break. Propose: echo the created expression's `index` (and/or `nodeId`) on the add_material_parameter response so add -> inspect -> connect compose deterministically on any material. Low — recoverable, works on a fresh material, but silently wrong on a non-empty one. Distinct from the readback stub (`B-material-get-node-details-missing-pins-props`) and from the MGIR-discoverability ticket (`E-material-mgir-bulk-path-undiscovered`); same creation-verb-no-node-id family as `E-add-event-no-node-id-echo` (blueprint, GUID-addressed).
