---
id: E-mandatory-node-position
title: "Make graph-node creation RPCs require explicit position"
status: DONE
severity: High
category: ergonomic
tags: [blueprint, material, animation, behaviortree, ergonomics, schema]
---

# Make graph-node creation RPCs require explicit position

Every `*.graph.create_node` / `*.add_*` method that places a node in a graph
declares `x` and `y` as `RPC_PARAM_OPT` and defaults missing values to 0.0.
Agents routinely omit coordinates, so every created node stacks at (0, 0)
and graphs become unreadable.

**Fix:** flip `x`/`y` to `RPC_PARAM_REQ` across all node-creating RPCs in
Blueprint, Material, Animation state machine, and Behavior Tree handlers.
Use `FHandlerContext::RequireNumber` for extraction. Rename
`blueprint.create_bpir_expression`'s `posX`/`posY` to `x`/`y` for parity.
Out-of-scope: `blueprint.add_function`, `blueprint.add_construction_script`,
`blueprint.insert_*` (anchor-relative), `niagara_graph.add_module`
(stack-implicit).

## History
- `#1-initial-report` `OPEN` reporter — "Node creation RPCs default x/y to 0; agents omit them → all nodes stack at origin. Promote to required across blueprint/material/animation/BT graph node creators; rename BPIR posX/posY to x/y."
- `#2-position-required-across-handlers` `IN-REVIEW` developer — "Flipped x/y RPC_PARAM_OPT→REQ across Blueprint, Material, Animation, BehaviorTree node-creating handlers; renamed BPIR posX/posY → x/y; bodies use Ctx.RequireNumber. See Wave-1 chunks 1A/1B/1C."
- `#3-verified-required-and-rejects` `DONE` tester — Verified: schema discovery shows `x` and `y` in the `required` list for `blueprint.graph.create_node`, `blueprint.create_bpir_expression`, `blueprint.graph.create_reroute_node`, and `material.authoring.add_material_node`. Live negative test: `blueprint.graph.create_node` called without `x` returns `{error:{code:"MISSING_REQUIRED_PARAM", message:"Missing required parameter 'x' (type: number)"}}`. Param descriptions explicitly say "required — nodes stack at origin if all callers pass 0". `posX`/`posY` rename confirmed (no `posX` or `posY` in the BPIR-expression schema).
