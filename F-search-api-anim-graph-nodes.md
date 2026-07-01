---
id: F-search-api-anim-graph-nodes
title: "No catalog / keyword search for AnimGraph node types"
status: DONE
severity: Medium
category: feature
tags: [search, animation, anim-graph, agir, node-discovery]
---

# No catalog / keyword search for AnimGraph node types

`UAnimGraphNode_*` subclasses are the building blocks of an Animation
Blueprint's AnimGraph — `UAnimGraphNode_SequencePlayer`,
`UAnimGraphNode_BlendSpaceEvaluator`, `UAnimGraphNode_StateMachine`,
`UAnimGraphNode_LayeredBoneBlend`, `UAnimGraphNode_LinkedAnimGraph`,
`UAnimGraphNode_TwoWayBlend`, `UAnimGraphNode_RotateRootBone`, plus
~120 others in stock UE 5.6. There is **no RPC that enumerates them**.

The story is the same as material expressions: each class IS the
operation (flat, single-tier — no K2Node-style wrapper indirection), but
the catalog of which classes exist is invisible to the agent. AGIR
(`call("anim")`) text-IR statements implicitly reference these classes,
and without a discovery surface the agent must already know every class
name to author or interpret AGIR.

**Use cases blocked:**

1. "I need a node that does X" — e.g. "additive layer", "speed-warp",
   "control rig evaluation", "blend by enum". Today: read engine source
   for `AnimGraphNode_*.h` (~120 files across `Engine/` and
   `Engine/Plugins/Animation/`), or trial-and-error class names.
2. AGIR generation from a high-level spec — an agent that knows it wants
   a state-machine root + several blend-space players cannot verify those
   classes exist without guessing.
3. Discovering required asset bindings — most `UAnimGraphNode_*` classes
   need a typed asset reference (`UAnimSequence`, `UBlendSpace`,
   `UAimOffsetBlendSpace`, `UAnimMontage`, `UPoseAsset`, `UAnimBlueprint`
   for sub-graphs). A bare class catalog isn't enough; the result row
   should expose which asset class(es) the node expects.

**Current workarounds:**

- `python.execute` enumerating `UAnimGraphNode_Base` subclasses via
  reflection.
- Read `Engine/Source/Editor/AnimGraph/Classes/AnimGraphNode_*.h` on disk.
- Grep AGIR examples / test matrix for known class names.

All bypass the typed-RPC surface.

**Proposal:** Add `animation.search_graph_nodes` (and
`animation.list_graph_nodes` for unranked enumeration), modeled on the
`material.graph` proposal in
[`F-search-api-material-expressions`](F-search-api-material-expressions.md)
but with anim-specific result fields:

```
animation.search_graph_nodes(
    query: string,                // keyword(s); ranked match on class name + category + description
    category?: string,            // restrict to AnimGraph categories ("Blends", "State Machine", "Sequence Player", "Linked Anim Graph", "Control Rig", "IK", ...)
    requiresAsset?: string,       // restrict to nodes that take this asset class (e.g. "BlendSpace", "AnimSequence")
    includeAbstract?: bool,       // default false
    limit?: number                // default 20
) -> {
    results: [{
        className: "AnimGraphNode_BlendSpacePlayer",
        runtimeNode: "FAnimNode_BlendSpacePlayer",   // FAnimNode_* struct it wraps at runtime
        category: "Blends",
        description: "Plays a Blend Space asset and outputs a sampled pose.",
        requiredAssets: ["BlendSpace"],              // typed asset(s) the node needs bound
        inputPins: [{ name: "BlendSpace", type: "BlendSpace" }],
        outputPins: [{ name: "Pose", type: "Pose" }],
        score: number
    }],
    totalMatches: number
}
```

Implementation surface: walk `TObjectIterator<UClass>` once filtered to
`UAnimGraphNode_Base` descendants. For each class, read the CDO's
`GetMenuCategory()` and `GetNodeTitle()` for category / description, then
peek the wrapped `FAnimNode_*` struct's `UProperty` set to compute pin
metadata and required-asset hints (most asset-binding nodes have a typed
`UPROPERTY` for their asset — e.g. `FAnimNode_BlendSpacePlayer::BlendSpace`).
Cache in-process; rebuild on hot-reload.

**Cross-ref:** Could be a thin wrapper around
[`F-search-api-native-uclasses`](F-search-api-native-uclasses.md) with
`parentClass="AnimGraphNode_Base"`, but the anim-specific result fields
(required asset class, wrapped `FAnimNode_*` struct, pin layout) make a
dedicated RPC more directly consumable by AGIR authoring callers. The
generic UClass search should still exist as a fallback.

**Notes:**

- Event Graph nodes in an AnimBP are plain K2Nodes — discovery for those
  is already covered by `blueprint.graph.list_node_types` /
  `blueprint.search_api`.
- AnimGraph nodes do **not** suffer the K2Node-style two-tier problem
  (one class = one operation), so no `target=` second arg is needed.

## History
- `#1-no-anim-graph-node-catalog` `OPEN` reporter — Surveying discovery RPCs across BP / material / Niagara / anim revealed `blueprint.graph.list_node_types` (K2Nodes) and `sequencer.list_track_types` (movie scene tracks) as the only typed catalog RPCs. No peer exists for `UAnimGraphNode_*` — ~120 classes invisible to the agent unless it already knows every name. AGIR authoring is currently blind to this catalog and must rely on training data or `python.execute`. Proposes `animation.search_graph_nodes` returning class name + wrapped `FAnimNode_*` struct + category + required asset class + pin layout, scoped via `parentClass` filter on `UAnimGraphNode_Base`. Complements [`F-search-api-native-uclasses`](F-search-api-native-uclasses.md) and [`F-search-api-material-expressions`](F-search-api-material-expressions.md).
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified gap by grep on `REGISTER_RPC_HANDLER("animation` across `Handlers/Animation/`: 49 `animation.authoring.*` + 12 `animation.*` registrations, all create/modify/inspect existing nodes (`add_state_machine`, `add_blend_node`, `set_anim_graph_node_value`, etc.). Zero discovery/catalog handlers for `UAnimGraphNode_*` classes. `blueprint.search_api` does not reach these (indexes UFunctions, not AnimGraph container classes). Cross-ref ticket `F-search-api-native-uclasses` is also still OPEN; even when shipped, it would only return class names + categories — the agent-actionable anim-specific fields (wrapped `FAnimNode_*` struct, `requiredAssets`, pin layout via FAnimNode UPROPERTY peek) justify a dedicated RPC layer atop the generic class search. No duplicates in `board/` (`F-anim-add-graph-node-generic` covers node *creation*, not class discovery). Severity Medium correct — Python reflection workaround exists but breaks unattended automation. Ticket stands as filed.
- `#3-shipped-anim-graph-catalog` `IN-REVIEW` developer — Added new file AnimGraphSearchHandler.cpp registering `animation.list_graph_nodes` and `animation.search_graph_nodes`. Live `TObjectIterator<UClass>` walk filtered to `IsChildOf(UAnimGraphNode_Base)` (no cache — matches existing `blueprint.graph.list_node_types` pattern). Per-entry: `className`, `runtimeNode` (FAnimNode_* struct name from `GetFNodeProperty()`), `category` (from CDO `GetMenuCategory`), `description` (CDO `GetNodeTitle`), `requiredAssets[]` (walking the FAnimNode struct's FObjectProperty fields whose PropertyClass derives from UAnimationAsset). Search ranks via simple substring-token scoring; empty query → INVALID_QUERY. Skipped detailed inputPins/outputPins arrays — would require AllocateDefaultPins on transient instances, out of scope for discovery. Regression test FAnimSearchGraphNodesCatalogTest covers list + search + empty-query rejection.
- `#4-verify-graph-node-search` `DONE` tester — Verified: `animation.search_graph_nodes` with `query:"blend space"` returned BlendSpace node rows including `className`, `runtimeNode`, `category`, `description`, `requiredAssets`, and `totalMatches:24`; `animation.search_graph_nodes` with `query:"player", requiresAsset:"AnimSequence"` returned `AnimGraphNode_SequencePlayer` with `runtimeNode:"FAnimNode_SequencePlayer"`; `animation.list_graph_nodes` with `limit:3` returned real AnimGraph node rows and `totalMatches:87`; empty search returned `INVALID_QUERY`.
