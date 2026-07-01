---
id: B-material-break-connections-named-pin-noop
title: "material.graph.break_connections does not actually disconnect on non-main nodes"
status: DONE
severity: High
category: bug
tags: [material, material-graph, break-connections, silent-failure]
---

# `material.graph.break_connections` is a silent no-op on non-main targets

`MaterialGraphHandler.cpp` `material.graph.break_connections` is supposed
to "Disconnect every wire on a material expression node, or just the named
pin". For the main material node it iterates the enumerated outputs (`BaseColor`,
`EmissiveColor`, etc.) and nulls the matching `Expression` pointer. For a
named target node, the flow is:

```cpp
UMaterialExpression* TargetExpr = FindExpressionByIdOrName(Material, NodeId);
if (TargetExpr)
{
    Material->PostEditChange();
    Material->MarkPackageDirty();
    // … returns SendSuccess …
}
```

**That's the entire branch.** No iteration of `FExpressionInputIterator`,
no `pinName` resolution, no `Input->Expression = nullptr`. The wires
remain intact. The RPC returns success.

The same call shape on `material.authoring.disconnect_nodes` works only
for the main node — there is no peer for "disconnect a single pin on a
mid-graph expression".

**Why it matters:**

- Surgical rewiring is the whole point of the low-level imperative
  surface. Today the only way to disconnect a mid-graph pin is to
  `remove_node` and re-add (losing the node's properties / parameter
  name / position), or to overwrite the input with a new
  `connect_nodes` call.
- The silent success makes this look like the operation worked. A caller
  that follows up with `compile_material` and `get_material_info` will see
  the same node count and same parameter list — exactly what a successful
  pin-only break should produce.

**Fix:**

When `TargetExpr` is found, walk `FExpressionInputIterator{TargetExpr}`:

- If `PinName` is non-empty, match `Expr->GetInputName(It.Index)` against
  `PinName` (with the same case-insensitive Equals used in
  `FindExpressionInputByName`) and set the matching `Input->Expression =
  nullptr`. Mind the function-call-input name decoration described in
  `B-material-function-call-input-name-decoration`.
- If `PinName` is empty, null every iterated `Input->Expression`.

Return a structured response: `{nodeId, pinsBroken: [...names], broken: true}`
so callers can verify what actually changed.

Also worth aligning naming: `material.authoring.disconnect_nodes` doesn't
support mid-graph pins at all (it only checks the main node). Either
forward to `material.graph.break_connections` with a fixed implementation,
or remove the misleading "Disconnect a pin from the main material node"
description and document the scope.

## Repro

1. Create a material `M`, add `Multiply` and a `Constant` source.
2. `material.graph.connect_nodes(materialPath: M, sourceNodeId: ConstId,
   targetNodeId: MulId, inputName: "A")`.
3. `material.graph.break_connections(materialPath: M, nodeId: MulId,
   pinName: "A")` — returns success.
4. `material.graph.get_node_details(nodeId: MulId)` — pin A still wired
   (currently `get_node_details` doesn't report this; cross-reference
   `B-material-get-node-details-missing-pins-props`). Open the material
   in the editor; the wire is still there.

## History
- `#1-break-connections-silent-noop` `OPEN` reporter — Material API audit caught `material.graph.break_connections` returning `SendSuccess` without touching `FExpressionInput` state when the target is a named non-main node. The branch in `MaterialGraphHandler.cpp` after `FindExpressionByIdOrName` does only `PostEditChange / MarkPackageDirty` then succeeds. The `pinName` parameter accepted by the RPC is completely ignored for non-main targets. Either implement `FExpressionInputIterator`-based clearing (named pin or all-pin variants) or report `NOT_IMPLEMENTED` so callers can fall back to `remove_node`.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified against `MaterialGraphHandler.cpp` lines 339–348: the non-main branch is exactly `FindExpressionByIdOrName` → `PostEditChange` → `MarkPackageDirty` → `SendSuccess`, with no iteration of pins and `PinName` unused. Severity High is correct (silent success is the worst failure mode for surgical rewiring, but workarounds exist via `remove_node`). Proposed fix via `FExpressionInputIterator` is the right shape; not literally one line — needs case-insensitive `GetInputName` match, mindfulness of the function-call-input decoration documented in `B-material-function-call-input-name-decoration`, and a branch for empty `PinName` that nulls every iterated input. No duplicate ticket found.
- `#3-iterate-and-clear-inputs` `IN-REVIEW` developer — Replaced the silent no-op non-main branch in MaterialGraphHandler.cpp material.graph.break_connections with: if PinName non-empty resolve via FMaterialExpressionFactory::FindExpressionInputByName (auto-inherits T2 raw-name handling) and null Input->Expression; else iterate FExpressionInputIterator and null every populated input, capturing each name. Response now returns {nodeId, broken: true, pinsBroken: [string]}. material.authoring.disconnect_nodes scoped out as a separate follow-up. Regression tests TestMaterialGraphBreakConnectionsNamedPin.cpp cover both named-pin and empty-pin paths on a Multiply expression.
- `#4-verify-fix` `DONE` tester — Verified: created /Game/__EA_Verify/M_McpVerifyBreakConnNamedPin, added Constant and Multiply nodes, connected Constant to Multiply.A, ran material.graph.break_connections(assetPath, nodeId=EB9B9E9D47953E1763A843BFA3A97E2C, pinName=A), observed broken=true, pinsBroken=["A"], and material.graph.get_node_details reported input A connectedNodeId="".
