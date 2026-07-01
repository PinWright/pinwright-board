---
id: F-anim-expose-pin-on-anim-node
title: "No RPC to toggle 'Expose as Pin' on AnimGraph node optional properties"
status: DONE
severity: High
category: feature
tags: [animation, anim-graph, imperative-api, optional-pin, variable-binding]
---

# No RPC to toggle 'Expose as Pin' on AnimGraph node optional properties

`UAnimGraphNode_*` classes carry their wrapped `FAnimNode_*` struct's
configurable inputs as **optional pins** — properties listed in
`ShowPinForProperties` (`TArray<FOptionalPinFromProperty>`). For each
entry the editor surfaces an "Expose as Pin" checkbox; flipping
`bShowPin=true` reconstructs the node so the property gains an input
graph-pin that can be wired to a `K2Node_VariableGet` / arithmetic
expression / pure function.

This is **the** mechanism for runtime binding on every player-style
node: `AnimGraphNode_SequencePlayer.PlayRate`, `.Sequence`,
`.bLoopAnimation`, `.StartPosition`; `AnimGraphNode_BlendSpacePlayer.X`,
`.Y`, `.PlayRate`; `AnimGraphNode_ApplyAdditive.Alpha`;
`AnimGraphNode_LayeredBoneBlend.BlendWeights`;
`AnimGraphNode_ControlRig.Alpha`, `.bAlphaBoolEnabled`,
`.AlphaCurveName`; etc. Without the flag the property stays a static
default and cannot reference a variable.

Confirmed empirically: `blueprint.graph.create_node` with
`nodeType="AnimGraphNode_SequencePlayer"` produces a node whose pin set
is `[{ Pose: Output PoseLink }]` only — no `PlayRate`, `Sequence`, or
`bLoopAnimation` pins exist until "Expose as Pin" is toggled. The
existing `animation.authoring.set_anim_graph_node_value` writes
`Node.X.Y` property defaults but cannot toggle structural pin
visibility.

**Use cases blocked:**

1. Binding a sequence player's `PlayRate` to a `SpeedMultiplier`
   variable (a foundational AnimBP pattern).
2. Binding a blend-space player's `X`/`Y` to gameplay-driven floats
   (Speed, Direction) — the entire reason BlendSpacePlayer nodes exist.
3. Binding `Alpha` on additive / layered / linked-anim nodes to a
   weight variable for blend control.
4. Authoring a complete AnimBP imperatively without falling back to
   AGIR for every node that needs a single variable binding.

**Workaround:** None confirmed. AGIR was initially assumed to handle
this via its node statements, but a source sweep of `Private/AGIR/`
(AGIRCompiler, AGIRParser, AGIRTextEmitter, AGIRPinResolver) finds no
`bShowPin` / `FOptionalPinFromProperty` / "expose" handling — AGIR
currently emits the same default pin set that `create_node` produces.
Until either AGIR grows expose-pin syntax or an imperative RPC ships,
variable binding on player-style nodes is unreachable via the MCP
surface. (Confirm by inspecting a hand-authored AnimBP where PlayRate
is exposed: round-trip through `anim.decompile_agir` and check whether
the exposed-pin state survives.)

**Proposal:** Add `animation.authoring.set_anim_graph_pin_exposed`
(params: `blueprintPath`, `graphName`, `nodeId` *or* `nodeName`,
`propertyName`, `exposed: bool`, `save?`). Resolves the
`UAnimGraphNode_Base*` by id, walks `ShowPinForProperties` for the
matching `PropertyName`, sets `bShowPin = exposed` and
`bCanToggleVisibility = true` if needed, then calls
`ReconstructNode()` so the new pin is materialized. After return the
caller can use `blueprint.graph.connect_pins` to wire the freshly
exposed pin to a variable getter.

Implementation surface: `UAnimGraphNode_Base::ShowPinForProperties` is
public; `UEdGraphNode::ReconstructNode()` is virtual and BLUEPRINTGRAPH_API.
The plugin already does similar `ReconstructNode()` cycles for cliff
families in the AGIR compiler (see `docs/anim` wiki, "AGIR cliff
completion" notes). A batch variant `set_anim_graph_pins_exposed`
(`pins: [{propertyName, exposed}]`) would let one reconstruct cover N
flips.

**Cross-ref:** Adjacent to the AGIR text-IR path
(`call("anim.compile_agir")`) but operates on the imperative surface so
single-pin tweaks don't require a full AGIR roundtrip. Pairs with
[`F-search-api-anim-graph-nodes`](F-search-api-anim-graph-nodes.md)
(discoverability) and a future asset-binding helper for player nodes.

## History
- `#1-no-expose-pin-rpc` `OPEN` reporter — `blueprint.graph.create_node` on `AnimGraphNode_SequencePlayer` produces a node with only the `Pose` output pin; the entire "Expose as Pin" mechanism for `PlayRate`/`Sequence`/`bLoopAnimation`/`StartPosition` (and equivalents on every other player/blender node) is invisible to the imperative API. `set_anim_graph_node_value` writes property defaults but can't toggle pin visibility, so variable binding via `connect_pins` is unreachable without falling back to AGIR. Proposes `animation.authoring.set_anim_graph_pin_exposed` (with batch variant) that walks `ShowPinForProperties` and calls `ReconstructNode()` to materialize the pin.
- `#2-reviewed-and-confirmed` `OPEN` tester — Confirmed via source sweep: (a) `blueprint.graph.set_node_property` in `BlueprintGraphHandler.cpp:2832` has a hardcoded allowlist (Comment/NodeComment/X/NodePosX/Y/NodePosY/bCommentBubbleVisible/bCommentBubblePinned) and cannot reach into `ShowPinForProperties[].bShowPin` — this is not a discovery/documentation problem, the generic API physically can't toggle it. (b) Zero matches for `bShowPin`/`ShowPinForProperties`/`FOptionalPinFromProperty`/`ExposeAsPin` across all `Handlers/Animation/` and `Private/AGIR/` source. (c) The original `#1` workaround claim that AGIR handles this is unverified — corrected the Workaround section to flag AGIR as also lacking expose-pin support pending a round-trip test. Severity High justified: variable binding on player nodes (PlayRate, BlendSpace X/Y, Alpha) is the most common AnimBP authoring pattern and is currently unreachable. No duplicate board entries; nearest siblings (`F-anim-bind-asset-on-player-node`, `F-search-api-anim-graph-nodes`) are adjacent concerns.
- `#3-shipped-expose-pin-rpcs` `IN-REVIEW` developer — Added `animation.authoring.set_anim_graph_pin_exposed` and batch `set_anim_graph_pins_exposed` in AnimationAuthoringHandler.cpp plus a shared `ToggleOptionalPinExposed` helper in AnimGraphConstructionUtils calling the engine's public `UAnimGraphNode_Base::SetPinVisibility` (rather than hand-rolled bShowPin+ReconstructNode, which would skip EvaluateOldShownPins). Regression test FAnimAuthoringSetGraphPinExposedTest in TestAnimGraphHandlers.cpp covers PlayRate exposure on a SequencePlayer.
- `#4-verify-expose-pin` `DONE` tester — Verified: created temp AnimBlueprint `/Game/App/UI/Test/W_McpVerifyTemp_F_anim_expose_pin_on_anim_node`, added `AnimGraphNode_SequencePlayer`, ran `animation.authoring.set_anim_graph_pin_exposed` for `PlayRate` with `exposed=true`, and `blueprint.graph.get_nodes` showed `PlayRate` as an Input pin plus `Pose` as Output; temp asset deleted afterward.
