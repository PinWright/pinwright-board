---
id: B-scs-get-local-child-parent-link-missing
title: "blueprint.scs.get omits parent link for locally-authored SCS children; child_count and parent disagree, scs.txt flattens the tree"
status: IN-REVIEW
severity: High
category: bug
tags: [scs, blueprint, components, readback, scs-txt, hierarchy]
---

# `blueprint.scs.get` can't express the hierarchy of a locally-authored SCS tree

When a Blueprint's component hierarchy is built entirely from local SCS nodes
(no inherited/native root), `blueprint.scs.get` emits the parent's
`child_count` but emits **no** `parent` field on the children, so the tree is
not reconstructable from the output and the downstream `scs.txt` renders every
node flat at the top level.

Root cause: `FSCSHandlers::GetBlueprintSCS`
(`EditorAutomationRpcGateway_SCSHandlers.cpp`) emits two relationship signals
that come from opposite ends of the SCS data model and disagree for local
children:
- `child_count` is `Node->GetChildNodes().Num()` (top-down — the parent's own
  `ChildNodes` array, line ~365).
- `parent` is emitted only `if (!Node->ParentComponentOrVariableName.IsNone())`
  (bottom-up, line ~343).

For a child attached under a **local** SCS root, UE stores the link only in the
parent's `GetChildNodes()`; the child's `ParentComponentOrVariableName` stays
`None` (that field is populated only when the parent is an inherited/native
component). So the parent reports `child_count: N` but the children carry no
`parent` field and no `children`/child-name list exists anywhere — the caller
sees "BeaconBase has 2 children" but cannot tell *which* of the other nodes
those two are, nor confirm any node's parent.

The defect propagates to `scs.txt`: `SCSTextEmitter::BuildText`
(`Handlers/Blueprint/SCSTextEmitter.cpp:221`) reconstructs the nested
`children {}` blocks **purely from the `parent` field**. With no `parent` on
local children, it renders all components flat — directly violating the
`blueprint.scs` wiki overlay's promise that `scs.txt` shows "nested
`children {}` sections reconstructed from parent links."

This is the locally-authored-root counterpart of the (DONE)
`B-asset-dump-scs-omits-inherited-parent`, which only fixed the case where the
parent is an *inherited* node (where `ParentComponentOrVariableName` IS set).
The `F-scs-dsl-sidecar` verification passed because its fixtures used inherited
roots, masking this case.

**Impact:** agents that build a beacon/prop class from scratch (the common
"add root + attach children under it" pattern) get a readback that misreports a
flat tree. In the seed repro the attempting agent burned several diagnostic
calls and an unnecessary `blueprint.scs.reparent_component` trying to reconcile
`child_count: 2` against the missing `parent`/flat `scs.txt`.

**Repro (live, confirmed):**
- `blueprint.scs.add_component` BeaconBase(StaticMeshComponent) → root,
  AlarmLight(PointLightComponent) → BeaconBase, DetectionZone(SphereComponent)
  → root (DetectionZone is re-parented under BeaconBase by UE root-validation).
- `call("blueprint.scs.get", {"blueprintPath":"/Game/Pickups/BP_WarningBeacon"})`
  returns:
  - `BeaconBase`: `"child_count": 2`, **no `parent`**
  - `AlarmLight`: `"child_count": 0`, **no `parent`** (was added under BeaconBase)
  - `DetectionZone`: `"child_count": 0`, **no `parent`**
- `call("asset.dump", {...})` → `scs.txt` renders `component(AlarmLight){...}`,
  `component(BeaconBase){...}`, `component(DetectionZone){...}` all at indent 0
  — no `children {}` nesting at all, despite BeaconBase's `child_count: 2`.

**Fix (proposed):** In `GetBlueprintSCS`, derive each node's parent from the
top-down structure too, not only `ParentComponentOrVariableName`. Concretely:
before the emit loop, build a child→parent map by walking every node's
`GetChildNodes()` (`Child->...` parent = the owning node's variable name), then
emit `parent` from that map when `ParentComponentOrVariableName` is `None`.
This makes `parent` consistent with `child_count` for local trees and lets the
existing `SCSTextEmitter` rebuild `children {}` correctly with no emitter
change. Alternatively (or additionally) emit an explicit `children: [names...]`
array so the relationship is expressible top-down without relying on the
bottom-up field.

## History
- `#1-initial-repro` `OPEN` reporter — Live repro on `/Game/Pickups/BP_WarningBeacon` (3 local SCS nodes: BeaconBase root, AlarmLight + DetectionZone under it). `blueprint.scs.get` returns `BeaconBase child_count:2` with no `parent` field on any node; `asset.dump`'s `scs.txt` renders all three flat (no `children {}`), contradicting both the `child_count` and the wiki overlay's "nested children reconstructed from parent links" guarantee. Root cause: `GetBlueprintSCS` emits `parent` only from `ParentComponentOrVariableName` (None for local-SCS children) while `child_count` comes from `GetChildNodes()`; `SCSTextEmitter` rebuilds nesting purely from the absent `parent`. Counterpart to the inherited-root-only `B-asset-dump-scs-omits-inherited-parent` (DONE). Fix: derive `parent` from each node's `GetChildNodes()` (or emit a `children` name array) so local trees are reconstructable.
- `#2-readback-parent-derivation` `IN-REVIEW` developer — Implemented the root-cause fix in `GetBlueprintSCS` (`Private/EditorAutomationRpcGateway_SCSHandlers.cpp`). Added a static `BuildScsChildParentMap(Nodes, OutChildToParent)` helper that walks each node's `GetChildNodes()` to recover the top-down child→parent link, then in the local-SCS loop (and the inherited-ancestor-SCS loop, per-ancestor-BP) emit `parent` from that map when `ParentComponentOrVariableName` is `None` — keeping the inherited/native case unchanged and making `parent` consistent with `child_count`. No `SCSTextEmitter` change needed: it already rebuilds `children {}` from `parent`. Regression test added: `Private/Tests/Blueprint/TestSCSGetLocalChildParentLink.cpp` (`FBlueprintScsGetEmitsParentForLocalChildrenTest`, "EditorAutomationRpcGateway.blueprint.scs.get.EmitsParentForLocalChildren") builds an all-local tree (BeaconBase root + AlarmLight + DetectionZone attached via `AddChildNode`, asserting the children's `ParentComponentOrVariableName` is `None` as the precondition), invokes the real `blueprint.scs.get` handler via `InvokeHandlerWithCapture`, and asserts each child emits `parent: BeaconBase` while the root emits none — failing if the derivation is reverted. Not yet compiled/tested (later phase).
