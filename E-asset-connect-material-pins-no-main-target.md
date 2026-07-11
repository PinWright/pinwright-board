---
id: E-asset-connect-material-pins-no-main-target
title: "asset.connect_material_pins only links expression->expression by index and can't target the Main material node's BaseColor/Roughness, so the asset.* add->wire flow can't be completed in-namespace — the caller must cross to material.authoring.connect_nodes"
status: OPEN
severity: Low
category: ergonomic
tags: [material, asset, connect-material-pins, main-node, cross-namespace, docs]
encounters: 1
lastSeen: 2026-07-11T13:02:39.2618058+03:00
---

# `asset.connect_material_pins` can't wire into the Main material node, so the asset.* material family is incomplete for add->wire

`asset.add_material_parameter` and `asset.connect_material_pins` live in the same
`asset.*` material family, and `connect_material_pins` exposes only
`fromExpression` (number), `toExpression` (number), and `inputName` — i.e. it
links **expression -> expression by numeric index** with **no way to target the
Main material node's** `BaseColor` / `Roughness` / other output inputs. So the
most natural next step after adding parameters — wire the Tint param into
BaseColor and the Roughness param into Roughness — **cannot be completed within
`asset.*`**. The caller has to switch namespaces to `material.authoring.connect_nodes`,
which accepts `targetNodeId:"Main"` for exactly that.

The sibling write RPC `material.authoring.connect_nodes` (and `material.graph.connect_nodes`)
documents and accepts the `"Main"`/empty sentinel for the main material node; the
`asset.*` connect verb has no equivalent, so the asset.* imperative surface is
internally incomplete for the last hop of an authoring flow.

## What it should do

Pick either:

- **Docs (cheap immediate win):** on the `asset.connect_material_pins` (and
  `asset.add_material_parameter`) per-method overlay in `docs/wiki-src/asset.md`,
  state that `connect_material_pins` links expression->expression only and that
  **main-node wiring** (BaseColor/Roughness/etc.) lives in
  `material.authoring.connect_nodes` (`targetNodeId:"Main"`) — or in the one-call
  `material.compile_mgir` bulk path. This lets an agent on the asset.* page reach
  the right verb without reading connect_material_pins.md, hitting its limits, and
  cross-navigating.
- **Capability (more complete):** give `asset.connect_material_pins` a way to
  target the Main node's inputs (the same `"Main"` sentinel + `inputName` its
  material.authoring cousin accepts), so the asset.* add->wire flow closes
  in-namespace.

## Why it's process friction (clean outcome, cross-namespace detour)

Surfaced in a clean `asset.add_material_parameter` task (focus
`asset.add_material_parameter`, namespace `asset`, outcome tool_bug for an
unrelated neighbor stub) authoring `/Game/Materials/M_EnvProp_Master`. After
adding the three params, the agent read the `asset.connect_material_pins` doc,
found it exposes only `fromExpression`/`toExpression`/`inputName` (expression pair
by index, no Main target), and crossed to `material.authoring.connect_nodes`
(`{sourceNodeId:"MaterialExpressionVectorParameter_0", targetNodeId:"Main",
inputName:"BaseColor"}` -> `{"message":"Connected to main material node."}`) to
wire Tint->BaseColor and Roughness->Roughness. Agent friction note verbatim:
*"asset.connect_material_pins (the asset.* family paired with add_material_parameter
indices) documents no way to target the main material node's BaseColor/Roughness,
so I used material.authoring.connect_nodes for the base-color/roughness wiring
instead."* The CallAnalyzer flagged the same gap (efficiency finding 3, pattern
"workaround"). The task still succeeded — the cost is the doc-read dead end plus
the namespace switch, not a failure.

Wiki page to improve: `docs/wiki-src/asset.md` (the `asset.connect_material_pins`
and `asset.add_material_parameter` overlay sections).

severity rationale: impact=friction (asset.* family structurally can't wire to
the Main node; a documented cross-namespace workaround exists) x reach=asset.*
legacy material authoring, an off-canonical path -> Low.

## Distinct from

- `E-material-connect-nodes-target-input-arg-asymmetry` (OPEN) — that is the
  `material.authoring.connect_nodes` *arg-name* asymmetry (`inputName` vs a guessed
  `targetInput`); this is that the *asset.* connect verb has no Main target at all.
  Different method, different root cause.
- `E-material-main-output-no-node-readback` (IN-REVIEW) — that is the *read* side
  (get_node_details rejecting the `"Main"` token); this is the *write* side of the
  asset.* family lacking a Main target. Different method, different half of the
  round-trip.
- `E-material-mgir-bulk-path-undiscovered` (IN-REVIEW) — that steers asset.*
  callers to the MGIR bulk path for the whole graph; this is the specific
  main-node-wiring gap in the single asset.* connect verb. Overlapping page, but
  a distinct docs edit / distinct capability.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean `asset.add_material_parameter` task (focus `asset.add_material_parameter`, namespace `asset`, outcome tool_bug for an unrelated neighbor stub) authoring `/Game/Materials/M_EnvProp_Master`. After adding three params via asset.*, the agent needed to wire Tint->BaseColor and Roughness->Roughness on the Main material node, but `asset.connect_material_pins` exposes only `fromExpression`/`toExpression`/`inputName` (expression->expression by numeric index) with no Main-node target — so the asset.* add->wire flow cannot be completed in-namespace. The agent read the connect_material_pins doc, hit the limit, and crossed to `material.authoring.connect_nodes` (`targetNodeId:"Main"`, `{"message":"Connected to main material node."}`) for both wires. Friction note verbatim: "asset.connect_material_pins (the asset.* family paired with add_material_parameter indices) documents no way to target the main material node's BaseColor/Roughness, so I used material.authoring.connect_nodes for the base-color/roughness wiring instead." CallAnalyzer flagged the same gap (efficiency finding 3, pattern "workaround"). Propose: docs minimum — note on the `asset.connect_material_pins`/`asset.add_material_parameter` overlay in `docs/wiki-src/asset.md` that main-node wiring lives in `material.authoring.connect_nodes` (`targetNodeId:"Main"`) or `material.compile_mgir`; optionally give asset.connect_material_pins a Main sentinel target. Low — clean outcome, documented cross-namespace workaround, off-canonical asset.* path. Distinct from `E-material-connect-nodes-target-input-arg-asymmetry` (arg name), `E-material-main-output-no-node-readback` (read side), and `E-material-mgir-bulk-path-undiscovered` (whole-graph MGIR steer).
