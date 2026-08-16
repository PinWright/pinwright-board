---
id: B-set-component-properties-staticmesh-shadow-corruption
title: "actor.set_component_properties wrote StaticMesh by raw FProperty store, leaving the engine's KnownStaticMesh shadow stale — an engine ensure on production HISM foliage, and a component that renders/streams/collides as the OLD mesh while the package serializes the new one"
status: DONE
severity: High
category: bug
tags: [actor, components, static-mesh, skinned-mesh, hism, reflection-write, engine-ensure, shadow-copy, silent-wrong-state, derived-state]
---

# `set_component_properties` bypassed `SetStaticMesh`, so the engine's shadow copy went stale

`actor.set_component_properties` applied **every** property with a raw `FProperty` store
(`ApplyJsonValueToProperty`). For `UStaticMeshComponent::StaticMesh` that is not a plain field:
the engine keeps a private shadow copy, `KnownStaticMesh`, that only its own setter updates. A
reflection write therefore left the shadow pointing at the **old** asset, and the handler's own
`UpdateComponentToWorld()` — three lines later — turned that divergence into an engine `ensure`.

The verb returned `applied: ["StaticMesh"]` throughout. The component then rendered, streamed and
collided as the old mesh while the package serialized the new one.

## Measured evidence

Raised on the production map's HISM foliage, verbatim from
`Utils/ComponentAssetPropertyWrite.h:15-21`:

```
Ensure condition failed: KnownStaticMesh == StaticMesh
StaticMesh property overwritten for component HierarchicalInstancedStaticMeshComponent
...HISM_Trees_Radiant_Conifer without a call to NotifyIfStaticMeshChanged().
```

Reproduced live on a probe HISM with the same stack and the same site —
`StaticMeshComponent.cpp:744` <- `UInstancedStaticMeshComponent::UpdateBounds` <-
`ComponentHandler.cpp`, with `AutoHandler_332_` two frames below the ensure
(`Tests/Actor/TestComponentAssetPropertyWrite.cpp:12-19`).

## Root cause, and the exact size of the class

The write site was `Handlers/Actor/ComponentHandler.cpp` in the property loop; the ensure fires
from the commit block immediately after it (`ComponentHandler.cpp:328-333`):

```cpp
  if (USceneComponent *SceneComponent = Cast<USceneComponent>(TargetComponent)) {
    SceneComponent->MarkRenderStateDirty();
    SceneComponent->UpdateComponentToWorld();
  }
```

Engine side, `UStaticMeshComponent::GetStaticMesh` (`StaticMeshComponent.h:453-463`) **repairs**
the divergence on the way past, so the stale state is not directly observable twice; the only
public signal is `OutdatedKnownStaticMeshDetected()`'s `ensureMsgf` opening at
`StaticMeshComponent.cpp:744`, and the only updater of the shadow is private
(`StaticMeshComponent.cpp:715-730`).

**The whole class was enumerated rather than guessed** — grepping the ensure text
`"without a call to Notify"` over `Engine/Source` yields exactly three shadow-copy properties
engine-wide (`Utils/ComponentAssetPropertyWrite.h:23-31`), each verified against UE 5.8 source:

| Property | Shadow | Engine site |
|---|---|---|
| `UStaticMeshComponent::StaticMesh` | `KnownStaticMesh` | `StaticMeshComponent.cpp:742-751` |
| `USkinnedMeshComponent::SkinnedAsset` | `KnownSkinnedAsset` | `SkinnedMeshComponent.cpp:6046-6052` |
| `UTexture::CompositeTexture` | `KnownCompositeTexture` | `Texture.cpp:1230-1235` |

The first two are reachable from the component verbs and are the two now routed. The third is an
**asset** property, unreachable from any component-property verb, and is already covered because
`property.set` calls `PostEditChange()`.

## Fix (shipped `88f6ee4f`)

- New `Utils/ComponentAssetPropertyWrite.{h,cpp}` owns the routing. `ApplyComponentAssetProperty`
  returns `Applied` / `Failed` / `NotApplicable`; the handler tries it first and falls through to
  the unchanged generic importer otherwise (`ComponentHandler.cpp:305-326`). The sibling
  `actor.add_component` had the identical defect and is routed the same way
  (`ComponentHandler.cpp:133-151`).
- `StaticMesh` -> `SetStaticMesh` (`ComponentAssetPropertyWrite.cpp:134`); `SkinnedAsset` ->
  `SetSkinnedAssetAndUpdate(Requested, /*bReinitPose*/ true)` (`:177`).
- **Not `PostEditChangeProperty`** — it is a subset of the setter, and synthesizing the chain form
  crashes on ISM (`InstancedStaticMesh.cpp:5638` dereferences the active member node with no null
  check).
- Every write is verified by **reading the component back** (`:141-148`, `:178-185`), because
  `SetStaticMesh` returns `false` both for "already this mesh" and for "refused". Clear is decided
  off the **request**, not off the stored value, so a wrong-class path is refused rather than
  silently erasing the mesh.
- On a HISM the cluster tree is rebuilt as the Details panel does
  (`BuildTreeIfOutdated(false, true)`, `:156-163`), and instances survive.

## Verification the write path cannot fake

`Tests/Actor/TestComponentAssetPropertyWrite.cpp` (528 lines, five tests) does **not** assert on
the ensure. The divergence is unobservable — `KnownStaticMesh` is private and every public reader
repairs it. What is observable is `UStaticMeshComponent::OnStaticMeshChanged()`
(`StaticMeshComponent.h:946`), broadcast by `SetStaticMesh` and by `PostEditChangeProperty` and by
**nothing else**; a raw reflection store broadcasts zero times. Counting that delegate is an exact,
deterministic proxy for "the engine's own change path ran", and it fails against pre-fix code for
the right reason instead of by catching a log line
(`TestComponentAssetPropertyWrite.cpp:21-29`).

Tests: `StaticMeshRunsEngineNotification` (:226), `HismStaticMeshKeepsInstances` (:281),
`StaticMeshRefusalsLeaveMeshIntact` (:334), `PerPropertyFailuresAreReported` (:411),
`AssetWriteRoutingBoundary` (:468).

## Related

- `B-set-component-properties-drops-warnings` — second defect in the same handler, found and fixed
  in the same commit, filed separately because the cause is unrelated.
- `E-actor-set-component-props-identifier-friction`, `E-property-route-no-component-path-discovery`
  — same verb, ergonomics.

## History
- `#1-ensure-on-production-hism` `OPEN` reporter — `actor.set_component_properties` returned `applied:["StaticMesh"]` and raised `Ensure condition failed: KnownStaticMesh == StaticMesh` / `StaticMesh property overwritten for component ...HISM_Trees_Radiant_Conifer without a call to NotifyIfStaticMeshChanged()` on the production map's HISM foliage. Root-caused to the handler applying every property through a raw `FProperty` store, so `UStaticMeshComponent::SetStaticMesh` never ran and the engine's private `KnownStaticMesh` shadow stayed on the old asset; the handler's own `UpdateComponentToWorld()` (`ComponentHandler.cpp:328-333`) then tripped `StaticMeshComponent.cpp:744`. Reproduced live on a probe HISM with the same stack and the same site, `AutoHandler_332_` two frames below the ensure. Durable consequence beyond the log line: the component renders, streams and collides as the OLD mesh while the package serializes the new one — silent wrong state behind an unqualified success.
- `#2-routed-through-the-engine-setters` `IN-REVIEW` developer — Fixed in `88f6ee4f`. New `Utils/ComponentAssetPropertyWrite.{h,cpp}` routes the shadowed properties to their engine setters and returns `Applied`/`Failed`/`NotApplicable` so the generic importer path is untouched for everything else (`ComponentHandler.cpp:305-326`); `actor.add_component` had the identical defect and is routed the same way (`:133-151`). **The class was enumerated, not sampled**: grepping `"without a call to Notify"` over `Engine/Source` gives exactly three shadow-copy properties engine-wide — `StaticMesh`/`KnownStaticMesh` (`StaticMeshComponent.cpp:742-751`), `SkinnedAsset`/`KnownSkinnedAsset` (`SkinnedMeshComponent.cpp:6046-6052`), `CompositeTexture`/`KnownCompositeTexture` (`Texture.cpp:1230-1235`), all three re-verified against UE 5.8 source. The first two are reachable from the component verbs and are routed to `SetStaticMesh` / `SetSkinnedAssetAndUpdate`; the third is an asset property no component verb can reach and is already covered because `property.set` calls `PostEditChange()`. Deliberately NOT `PostEditChangeProperty`: it is a subset of the setter and synthesizing the chain form crashes on ISM (`InstancedStaticMesh.cpp:5638` dereferences the active member node unchecked). Each write is verified by reading the component back, because `SetStaticMesh` returns false both for "already this mesh" and for "refused"; clear is decided off the request rather than off the stored value, so a wrong-class path is refused instead of silently erasing the mesh. HISM cluster trees are rebuilt as the Details panel does and instances survive.
- `#3-verified` `DONE` tester — Verified by a test that cannot pass on the pre-fix code for the wrong reason. `KnownStaticMesh` is private and every public reader repairs the divergence on the way past (`StaticMeshComponent.h:453-463`), so the stale state cannot be observed twice; the tests instead count `UStaticMeshComponent::OnStaticMeshChanged()` (`StaticMeshComponent.h:946`), which `SetStaticMesh` and `PostEditChangeProperty` broadcast and nothing else does — a raw reflection store broadcasts zero times, making the delegate count an exact deterministic proxy for "the engine's own change path ran" (`Tests/Actor/TestComponentAssetPropertyWrite.cpp:21-29`). Five tests, 528 lines: `StaticMeshRunsEngineNotification`, `HismStaticMeshKeepsInstances`, `StaticMeshRefusalsLeaveMeshIntact`, `PerPropertyFailuresAreReported`, `AssetWriteRoutingBoundary`. Suite reported by the fix commit as **3795 performed / 3795 pass / 0 fail** (baseline 3789 + 6 new), queue drained, 0 ensures. Filed retroactively: the defect went discovery -> fix -> verification inside one working day and would otherwise have left no board record that it ever existed.
