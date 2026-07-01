---
id: E-material-list-all-omits-position
title: "No single-call readback returns every material node's (x,y): get_node_details list-all omits position and get_material_info has no per-node coords, so verifying a layout costs one get_material_node_details call per node"
status: WONTFIX
severity: Medium
category: ergonomic
tags: [material, material-graph, material-authoring, get-node-details, get-material-info, auto_layout, readback, list-mode, missing-field, excessive-steps]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Reading back every node's position takes N singular `get_material_node_details` calls — no list-all / info surface carries (x,y)

Verifying node positions after an authoring or `auto_layout` pass — "did every
node actually move off (0,0) to a distinct spot?" — has **no single-call route**
on the material surface. Every graph-wide readback either omits `nodeId`-keyed
positions or omits positions entirely, so the only way to read N node positions
is N singular `get_material_node_details` calls (one per node):

- **`material.authoring.get_material_node_details`** has **no list mode** — `nodeId`
  is `RPC_PARAM_REQ` (`MaterialAuthoringHandler.cpp` ~L3032/L3038). To read 5 node
  positions you call it 5 times. The single-node payload it returns
  (`MGIRExpressionUtils::BuildExpressionDetailsJson`) **does** carry `x`/`y`
  (`MGIRExpressionUtils.h:205-206`) — so the data is right there per node, just
  never available in bulk.
- **`material.graph.get_node_details`** with `nodeId` omitted *does* list all nodes
  (the documented list-all mode), but each `availableNodes` entry is built as only
  `{nodeId, nodeType, index, desc?}` (`MaterialGraphHandler.cpp:368-374`) — it
  **drops `x`/`y`**, even though the method's own summary advertises it returns
  *"detailed info (class, **position**, pin connectivity, ...)"* (L319). The
  list-all branch and the single-node branch (`BuildExpressionDetailsJson`,
  L356) diverge on exactly the one field a layout-verify needs.
- **`material.authoring.get_material_info`** returns
  `{domain, blendMode, twoSided, shadingModel, nodeCount, mainInputs[], parameters[]}`
  (`MaterialAuthoringHandler.cpp` ~L2740-2790) — `nodeCount` but **no per-node
  positions** at all.

So the natural call after `material.authoring.auto_layout` (whose entire purpose
is to reposition nodes — its response reports `expressionsLaidOut=N` but not the
resulting coordinates) is the singular `get_material_node_details`, once per node,
to confirm the re-flow worked. This is the taxonomy's "N calls where a single
batch should exist" — except here the batch surface (`get_node_details` list-all)
already exists and is even documented to carry position; it just silently omits
the field, so it can't serve the intent.

## What it should do

Smallest fix (no new verb, no behavior change beyond an added field): in the
`material.graph.get_node_details` list-all branch, set `x`/`y` on each
`availableNodes` entry from `Expr->MaterialExpressionEditorX/Y` (the same values
`BuildExpressionDetailsJson` already emits for the single-node path). That makes
the documented "position" claim true and turns the whole layout-verify into one
call. Optionally mirror a compact `nodes:[{nodeId, x, y}]` (or a list-all mode)
onto `material.authoring.get_material_node_details` / `get_material_info` so the
authoring-namespace readback is symmetric and an agent doesn't have to cross to
`material.graph` to bulk-read positions. Docs: note in
`docs/wiki-src/material.authoring.md` that the one-call positions readback is
`material.graph.get_node_details` (no `nodeId`), and that `auto_layout`'s result
reports only a count — read positions back via that list-all call, not per node.

This is distinct from:
- `B-material-get-node-details-list-mode-errors` (IN-REVIEW) — that fixes the
  list-all *envelope* (it was wrongly returned as `NODE_NOT_FOUND` instead of
  success); it does not add the missing `x`/`y` field to the entries.
- `E-get-node-details-batch-undiscovered-on-read-path` (OPEN) — that is the
  **blueprint** namespace, where a working `get_node_details_batch` exists and the
  gap is pure discoverability; here the material batch surface *exists and is
  reached* but **drops the position field**, a missing-field gap, not discovery.
- `E-material-main-output-no-node-readback` (IN-REVIEW) — Main-node sentinel
  inspectability + shadingModel, unrelated to per-node coordinates.
- `F-graph-layout-report-rpc` (OPEN, blocked on `F-graph-layout-metrics-core`) —
  a heavyweight layout-*quality* metrics report (overlap/tangle scores); overkill
  and gated for the plain "read back the coordinates" need this ticket covers.

## Evidence

mcp-test-workflow process audit of a `material.authoring.auto_layout` task on
`/Game/Materials/M_WetRoad` (19 calls, outcome **clean**, self-reported friction
**"none"**). After `auto_layout` reported `expressionsLaidOut=5`, the success
check ("verify the nodes actually moved to distinct, non-overlapping positions")
was performed with **five sequential singular `get_material_node_details` calls**,
one per node:

- `get_material_node_details` — *"Roughness (post) -> 320,180"*
- `get_material_node_details` — *"Wetness (post) -> 320,720"*
- `get_material_node_details` — *"TilingScale (post) -> 320,900"*
- `get_material_node_details` — *"TextureCoordinate (post) -> 320,540"*
- `get_material_node_details` — *"Multiply (post) -> 320,360"*

(plus 2 more singular `get_material_node_details` pre-layout to confirm the (0,0)
pile-up). All five node ids were known and the intent was uniform ("show me every
node's new x/y"); a single list-all readback would have replaced the five round
trips with one — but no list-all/info surface returns positions, so the per-node
fan-out is the only route. No error, no retry, no misuse-then-correct, so it never
surfaced as friction in the report — exactly why it slips through: a
clean-but-N-round-trips-too-long readback that repeats on every
author/auto_layout-then-verify-positions material task. (`get_material_info` was
also called twice in the task but only yields `nodeCount`/params, never the
coordinates the verify needs.)

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of the `material.authoring.auto_layout` M_WetRoad task (19 calls, outcome clean, friction self-reported "none"): verifying that auto_layout moved 5 nodes off (0,0) cost 5 singular `get_material_node_details` calls (one per node) because there is no single-call positions readback. Root cause is a missing field, not discovery: `material.graph.get_node_details` list-all mode builds each `availableNodes` entry as `{nodeId,nodeType,index,desc?}` and drops `x`/`y` (`MaterialGraphHandler.cpp:368-374`) even though its summary advertises "position", while `material.authoring.get_material_node_details` has no list mode (nodeId required) and `get_material_info` carries no per-node coords. The single-node `BuildExpressionDetailsJson` already emits `x`/`y` (`MGIRExpressionUtils.h:205-206`), so the data exists per node. Propose: add `x`/`y` to the list-all `availableNodes` entries (cheapest — makes the documented "position" true and the verify one call), optionally mirror a bulk `nodes:[{nodeId,x,y}]` onto the authoring readback, and doc the one-call positions route in material.authoring.md. Distinct from B-material-get-node-details-list-mode-errors (list-all envelope, not the missing field), E-get-node-details-batch-undiscovered-on-read-path (blueprint discoverability of a working batch), E-material-main-output-no-node-readback (Main sentinel), and F-graph-layout-report-rpc (heavyweight gated metrics).
- `#2-attempt-failed` `OPEN` developer — Auto-fix attempt reached NO-RESULT; reverted and NOT pushed (build/tests not green).
- `#3-wontfix` `WONTFIX` developer — Declined on the worth-it lens. The asymmetry is real (list-all `availableNodes` entries at `MaterialGraphHandler.cpp:368-374` build `{nodeId,nodeType,index,desc?}` and drop x/y; single-node `BuildExpressionDetailsJson` emits them — `MGIRExpressionUtils.h:205-206`), but the ticket's "documented-contract-violation" framing is a misreading: the L319 summary parenthetical "(class, position, pin connectivity, parameter name)" grammatically qualifies "for one node" (the single-node path, which DOES deliver x/y), while the list-all clause promises only "list every node in the graph when nodeId is omitted" — enumeration, which it delivers (success at L385). So this is a plain enrichment ask, not a make-the-docs-true correctness fix. Sole evidence is ONE `auto_layout` task on M_WetRoad, outcome clean, self-reported friction "none" — friction inferred by auditing an already-successful task (N redundant read round-trips, zero agent-perceived pain), the exact evidentiary profile the board already declined for the blueprint analogue `E-get-node-details-batch-undiscovered-on-read-path` (WONTFIX, `#2-wontfix`: "the textbook cosmetic discoverability nit ... zero observed friction ... Not worth the churn"). Consistency demands the same disposition; the `#2-attempt-failed` NO-RESULT auto-fix is added churn against zero friction. If a real author/auto_layout-then-verify-positions task later surfaces genuine friction, reopen with that evidence.
