---
id: F-anim-layered-blend-bone-weights
title: "add_layered_blend_per_bone takes only one bone — no multi-layer weights, branch filters, or per-bone weight authoring"
status: DONE
severity: High
category: feature
tags: [animation, anim-graph, layered-blend, bone-mask, imperative-api]
---

# `add_layered_blend_per_bone` does not author the real LayerSetup

`animation.authoring.add_layered_blend_per_bone` exposes a single
`boneName` parameter. The wrapped runtime node
`FAnimNode_LayeredBoneBlend` actually drives blending from
`TArray<FInputBlendPose> LayerSetup` — each layer carries a
`TArray<FBranchFilter>` of `{ BoneName, BlendDepth }` pairs that define
the mask region. Plus there's `BlendMode` (`BranchFilter` vs
`BlendMask`), an optional `BlendMask` reference
(`UAnimBoneMaskAsset*`), `bMeshSpaceRotationBlend`,
`bMeshSpaceScaleBlend`, `CurveBlendOption`, and the per-layer alpha
inputs surfaced via "Expose as Pin" (see
[`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md)).

A single-bone API can author one trivial mask. Real AnimBP layered
blends mask several bones at once (e.g. upper-body additive: spine_01
+ clavicle_l + clavicle_r, each with their own depth), use a typed
`UAnimBoneMaskAsset` (UE 5.4+ workflow), or layer 3+ poses with
independent masks. None of that is reachable through this RPC, and
the wrapped struct's fields cannot be set via
`set_anim_graph_node_value` because `LayerSetup` is a
`TArray<FInputBlendPose>` (nested struct array) — the string-typed
property setter doesn't reliably round-trip nested struct arrays.

**Use cases blocked:**

1. Multi-bone upper-body additive layer (the canonical use of this
   node).
2. `BlendMode=BlendMask` workflows using a `UAnimBoneMaskAsset`
   reference.
3. Per-layer `BlendDepth` authoring (negative depth = include children,
   non-negative = stop at depth N — defines the actual mask shape).
4. Adding more than one layer pose (the runtime node supports N input
   poses; the helper creates only one).

**Workaround:** Compile via AGIR
(`call("anim.compile_agir")`) — the text-IR walks
`UAnimGraphNode_LayeredBoneBlend` exhaustively. The imperative path is
underbuilt.

**Proposal:** Add `animation.authoring.set_layered_blend_layers`
(params: `blueprintPath`, `graphName`, `nodeId`, `layers`,
`blendMode?`, `blendMaskPath?`, `meshSpaceRotationBlend?`,
`meshSpaceScaleBlend?`, `curveBlendOption?`, `save?`) where `layers`
is `[{ branchFilters: [{ boneName, blendDepth }], blendWeight?:
number }]`. Resolves the node, rewrites `LayerSetup` from the array,
calls `ReconstructNode()` so the per-layer pose pins repopulate, then
marks the BP structurally modified.

Also extend `add_layered_blend_per_bone` to accept `layers:
[{branchFilters: [...]}]` in place of `boneName` so simple multi-bone
authoring doesn't need a follow-up call. Keep `boneName` as a
deprecated single-layer shortcut.

Implementation surface: `UAnimGraphNode_LayeredBoneBlend::Node` is
public; `FAnimNode_LayeredBoneBlend::LayerSetup` is editable. The same
`ReconstructNode()` + `MarkBlueprintAsStructurallyModified` cycle the
AGIR cliff handlers use already applies. The bone-mask asset path
resolves through `StaticLoadObject<UAnimBoneMaskAsset>` (UE 5.4+; the
class is BLUEPRINTGRAPH_API or its parent ENGINE_API).

**Cross-ref:** Pairs with
[`F-anim-expose-pin-on-anim-node`](F-anim-expose-pin-on-anim-node.md)
for binding per-layer `BlendWeight` to variables. Similar gap on
`AnimGraphNode_LayeredBoneBlendByAsset` if it exists.

## History
- `#1-single-bone-only` `OPEN` reporter — `add_layered_blend_per_bone` takes one `boneName`; the runtime node `FAnimNode_LayeredBoneBlend.LayerSetup` actually drives blending from a `TArray<FInputBlendPose>` of per-layer `TArray<FBranchFilter>` `{BoneName, BlendDepth}` entries plus `BlendMode` / `BlendMask` / `bMeshSpaceRotationBlend` / `CurveBlendOption`. Multi-bone masks, blend-mask-asset mode, and multi-layer authoring are all unreachable through the imperative API and can only be expressed via AGIR text-IR. Proposes `set_layered_blend_layers` rewriting `LayerSetup` + `ReconstructNode()` and extending `add_layered_blend_per_bone` to accept a `layers` array.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified at `AnimationAuthoringHandler.cpp:2750-2809`. Worse than originally reported: the `boneName` param is **declared and read into a local but never used** — the handler only calls `FGraphNodeCreator<UAnimGraphNode_LayeredBoneBlend>`, sets `NodePosX/Y`, and finalizes. Inline comment at L2793-2794 admits "Configuring specific bone layers requires access to BlendNode->Node.LayerSetup which is typically done through the editor UI." So even the trivial single-bone case isn't authored — the imperative path creates an empty node. Severity High is justified (canonical use case of this node is multi-bone upper-body additive masking, completely unreachable). AGIR (`anim.compile_agir`) walks `UAnimGraphNode_LayeredBoneBlend` exhaustively and is the working path today; ticket already documents this as the workaround. Recommend: keep AGIR as the primary recommendation for layered-blend authoring in the wiki, and treat this imperative gap as a lower-priority parity follow-up (existing API extension with `layers` array, per ticket proposal) rather than a P0. No board duplicates; ticket is accurate.
- `#3-shipped-layered-blend-layers` `IN-REVIEW` developer — Added `animation.authoring.set_layered_blend_layers` in AnimationAuthoringHandler.cpp plus a `WriteLayeredBlendLayers` helper in AnimGraphConstructionUtils that inlines the LayerSetup/BlendPoses/BlendWeights/BlendMasks lock-step grow (engine's `AddPose`/`SyncBlendMasksAndLayers` aren't DLL-exported in 5.6 — pattern copied from AGIRCompiler_LayeredBlend.cpp). Extended existing `add_layered_blend_per_bone` to accept `layers` and to actually USE the `boneName` param. BlendMask asset binding routes through `UBlendProfile` (not `UAnimBoneMaskAsset` which doesn't exist in 5.6). Regression test FAnimAuthoringSetLayeredBlendLayersTest authors a 2-layer mask (spine_01/-1 + clavicle_l/0; neck_01/1) and asserts LayerSetup.Num()==2 with correct branch filter contents.
- `#4-verify-layered-blend-layers` `DONE` tester — Verified: live `system.run_tests` with `tests:["EditorAutomationRpcGateway.anim.authoring.SetLayeredBlendLayers"]` completed with `has_errors:false`, resolved the requested test, and reported no missing tests; the live `call("animation.authoring.set_layered_blend_layers")` wiki exposes the expected layered-blend parameters, and the covered test asserts the 2-layer `LayerSetup` branch-filter payload with `spine_01/-1`, `clavicle_l/0`, and `neck_01/1`.
