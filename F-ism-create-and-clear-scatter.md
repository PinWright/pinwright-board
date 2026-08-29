---
id: F-ism-create-and-clear-scatter
title: "Three typed verbs read, write and ground the instances of an ISM/HISM scatter and no verb creates or empties one — AddInstance / RemoveInstance / ClearInstances have zero call sites in the plugin, so the only way to populate the array the other three operate on is object.call_function per instance"
status: OPEN
severity: Medium
category: feature
tags: [actor, spatial, ism, hism, instanced-static-mesh, scatter, missing-verb, create, clear, add-instance, remove-instance, lifecycle-gap, vegetation]
encounters: 1
lastSeen: 2026-08-29
---

# The scatter lifecycle ships its middle and neither end

Three typed verbs operate on a plain (non-foliage) ISM/HISM instance array:

- `actor.get_instances` — `Plugins/PinWright/Source/PinWright/Private/Handlers/Actor/InstancedMeshHandler.cpp:191`
- `actor.set_instance_transforms` — `InstancedMeshHandler.cpp:291`
- `spatial.ground_instances` — `Handlers/Spatial/GroundPlacementHandler.cpp:1235`

Read, write, solve. Nothing creates the array they address, and nothing empties it. A caller who
does not already have a scatter cannot make one, and a caller who has a wrong one cannot discard it.

**Confirmed absent, mechanically.** `UInstancedStaticMeshComponent::AddInstance`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/Components/InstancedStaticMeshComponent.h:271`),
`AddInstances` (`:275`), `RemoveInstance` (`:417`), `RemoveInstances` (`:421`, `:427`) and
`ClearInstances` (`:431`) have **zero** call sites anywhere under
`Plugins/PinWright/Source/PinWright/Private/Handlers/` and `.../Utils/`. The only `AddInstance` /
`RemoveInstances` hits in the handler tree are `FFoliageInfo::AddInstance` / `::RemoveInstances`
(`Handlers/Environment/FoliageHandler.cpp:468`, `:481`, `:498`, `:1228`, `:1232`) — a different API
on a different object, operating on the foliage ledger inside an `AInstancedFoliageActor` — and
`AActor::AddInstanceComponent` (`Handlers/Actor/ComponentHandler.cpp:96`,
`Handlers/Actor/ComponentDuplicateHandler.cpp:144`,
`Handlers/Environment/EnvironmentHandler.cpp:878`, `Handlers/Geometry/SplineHandler.cpp:270`,
`:1099`, `:1227`), which attaches a component to an actor and has nothing to do with instances.

## The evidence is inside the parent ticket, and both halves check out

`F-ism-per-instance-transforms` (IN-REVIEW, High) names this exact absence — as evidence for its own
case, not as something it intended to close. Quoting its body verbatim:

> **Zero calls to `GetInstanceTransform` exist anywhere in the tree, tests included.** Every
> non-test `AddInstance` / `UpdateInstanceTransform` / `RemoveInstance` /
> `BatchUpdateInstancesTransforms` call site is absent; the only `AddInstance` hits outside
> `Private/Tests/**` are `FFoliageInfo::AddInstance` in the foliage handler, which is a different
> API on a different object.

Its `## Proposed verb shape — three verbs, not one` section then proposes, verbatim:

> 1. **`actor.get_instances`** — `{actorName, component?, indices?, limit?, offset?,
>    space:"world"|"local"}` → `[{index, location, rotation, scale}]` as decomposed transforms, plus
>    `instanceCount`.
> 2. **`actor.set_instance_transforms`** — `{actorName, component?, instances:[{index, location?,
>    rotation?, scale?}], space, expectedCount?}`.
> 3. **`spatial.ground_instances`** — `{actorName, component?, indices?, surface (REQUIRED, same
>    schema as `ground_actors`), samples, seatPercentile, embed, apply:true}`.

Three verbs, none of which creates or removes an instance. `AddInstance` and `RemoveInstance` are
cited in the diagnosis and then absent from the proposal — deliberately out of scope, not proposed
and dropped. Its `#2-three-verbs-shipped` entry confirms all three landed *"in the proposed shape"*,
so nothing widened on the way in. This ticket asks for the two ends that were never asked for.

## The partial escape hatch, named honestly, because a reviewer will find it

**A caller can create the component.** `actor.add_component` takes an arbitrary `componentType`
gated only on `ComponentClass->IsChildOf(UActorComponent::StaticClass())`
(`ComponentHandler.cpp:75-77`) and constructs it with `NewObject<UActorComponent>(Found,
ComponentClass, ...)` (`:87-88`), so
`componentType: "HierarchicalInstancedStaticMeshComponent"` yields a real, attached, empty HISM.

**And the mesh comes with it — the documentation on this point is wrong, in the caller's favour.**
The declared parameter text says *"Asset path of a static mesh; only used when componentType is
StaticMeshComponent"* (`ComponentHandler.cpp:47`, rendered verbatim to
`Saved/PinWright/wiki/actor.add_component.md` under **Parameters**, and restated in that page's
`## Notes` as *"`StaticMeshComponent`s auto-load `meshPath`"*). The code does not test for that
class — it tests for assignability:

    if (UStaticMeshComponent *SMC = Cast<UStaticMeshComponent>(NewComponent)) {
      FString MeshPath = Ctx.GetString(TEXT("meshPath"));

`ComponentHandler.cpp:113-114`. `UHierarchicalInstancedStaticMeshComponent` derives from
`UInstancedStaticMeshComponent` derives from `UStaticMeshComponent`, so the cast succeeds and
`SetStaticMesh` runs (`:118`). The `property.set` follow-up the docs imply is necessary is not.
**A fixer should correct that parameter text in the same commit**; it is currently steering callers
into an extra RPC they do not need, and it understates the one escape hatch that exists.

**What no route reaches is the instance array itself.** Having created an empty HISM with its mesh
set, there is still nothing typed that puts a transform into it or takes one out.
`actor.set_instance_transforms` writes existing indices only — its pre-flight refuses the whole
batch on a stale index (`F-ism-per-instance-transforms` `#2`: *"one stale index, one duplicated
index, or an `expectedCount` disagreeing with `GetInstanceCount()` refuses the whole batch before
the first write"*), which is correct behaviour for a write verb and is exactly why it cannot double
as a create verb. So the shipped read/write/solve trio is inert on a component the caller can build
but not fill.

**The untyped route works and is undocumented.** `AddInstance`, `AddInstances`, `RemoveInstance`,
`RemoveInstances` and `ClearInstances` are all `UFUNCTION(BlueprintCallable)` at the header lines
above, and they live on the live component, not on a CDO — so `object.call_function` reaches them
(its one blanket refusal is `RF_ClassDefaultObject` targets,
`Handlers/Reflection/ObjectCallFunctionHandler.cpp:107-110`, which does not apply). That is a
source dive: nothing in the generated wiki mentions it, and `AddInstance` costs one RPC per
instance, which for a scatter is the whole point of the batch verbs that already shipped beside it.

## Proposed verb shape

Two verbs, matching the naming and the batch shape of the three that shipped:

1. **`actor.add_instances`** — `{actorName, component?, meshPath?, transforms:[{location, rotation?,
   scale?}], space:"world"|"local", createComponent?}` → `{addedIndices[], instanceCount}`.
   Routes through `AddInstances(..., bShouldReturnIndices=true, ...)` so the caller gets the indices
   the other three verbs address, in one call rather than N. `createComponent` covers the
   create-empty-then-fill case in one round trip instead of chaining `actor.add_component`.
2. **`actor.remove_instances`** — `{actorName, component?, indices?, all?, expectedCount?}` →
   `{removed, instanceCount, removedInstances[]}`. `indices` omitted with `all:true` routes through
   `ClearInstances`; otherwise `RemoveInstances`. `expectedCount` mirrors
   `actor.set_instance_transforms`' guard for the same reason: a scatter is shared level state and
   an index list is not a scope.

Three points a fixer needs, all inherited from the shipped verbs rather than new:

- **Index invalidation is the hazard that has no analogue in the write verb.** `RemoveInstance`
  swap-removes, so every index a caller holds from `actor.get_instances` may name a different
  instance afterwards. `removedInstances[]` must echo enough for the caller to re-anchor, and the
  doc must say the indices moved — `actor.set_instance_transforms` never had to.
- **Reuse `InstancedMeshUtils::FinishInstanceWrites`** (`Handlers/Actor/InstancedMeshUtils.h`) for
  the one-per-batch `MarkRenderStateDirty` → HISM `BuildTreeIfOutdated(false, true)` under
  `FApp::CanEverRender()` → `MarkPackageDirty` sequence, exactly as `spatial.ground_instances` does
  at `GroundPlacementHandler.cpp:1541-1544`. A create/clear changes the cluster tree at least as
  much as a move does.
- **Carry the no-`FScopedTransaction` decision forward, or overturn it deliberately.**
  `F-ism-per-instance-transforms` `#2` records it (`Modify()` on an ISM serialises the whole
  `PerInstanceSMData` array per call) and leans on the echoed pre-write transforms as the undo of
  record. That reasoning transfers to add (the undo is a remove) but **not** to clear: nothing the
  response can echo reconstructs a discarded scatter cheaply, so `all:true` is the one seat that may
  need the transaction the other verbs decline. Decide it explicitly.

**Severity: Medium, argued.** The impact class looks like the rubric's *"High or Medium: hard
blocker with no workaround (a stub, a missing verb, or rejecting valid input)"* — two missing verbs
— and it is the lower of the two, because the workaround is real rather than theoretical:
`object.call_function` reaches all five BlueprintCallable entry points on the live component, and
`ClearInstances` in particular is a single no-argument call. That lands on the rubric's **Medium**
verbatim — *"Doable, but only via a documented workaround, a source dive, or many extra calls"* —
and it is all three at once (undocumented in the wiki, found only by reading the engine header, and
one RPC per instance on the add path). It is not High: nothing is silently wrong here. Every verb
that exists reports honestly; a caller pointed at an empty HISM gets `instanceCount: 0` from
`actor.get_instances`, not a lie. **Reach modifier declined in both directions, and named:** the
bump up would need this to run in almost every session, and instanced scatters do not; the bump down
would need it to be a rare edge path, and it is not — `spatial.ground_actors`' `HOLDER_NOT_SEATABLE`
refusal (`GroundPlacementHandler.cpp:718-720`) makes the ISM/HISM scatter a first-class object this
plugin routes callers toward. Medium stands unmodified.

## Related

- `F-ism-per-instance-transforms` (IN-REVIEW, High) — the parent. Read/write/solve shipped; its
  quoted proposal section is the evidence that create/clear were scoped out rather than missed.
- `B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) — `foliage.remove` empties
  `FFoliageInfo::Instances` and never touches the HISM, so the foliage namespace's remove path is
  ledger-only and its verification verb reads the same emptied ledger. Part of why a caller who
  wants a scatter they can actually clear reaches for a plain HISM instead of the foliage system —
  and a warning about what a `remove` verb on this side must not do.
- `F-scatter-layout-verb` (DONE) — `spatial.scatter_layout` produces the transform set with nothing
  typed to put it in. That verb's output is precisely `actor.add_instances`' input.
- `B-ism-undo-record-unsafe` (OPEN, High) — the space-marker contract any new `movedInstances[]` /
  `removedInstances[]` echo must satisfy. Do not ship a fourth echo with the same defect.

## History
- `#1-no-create-or-clear-verb` `OPEN` reporter — Confirmed zero call sites for
  `UInstancedStaticMeshComponent::AddInstance` / `AddInstances` / `RemoveInstance` /
  `RemoveInstances` / `ClearInstances` across `Private/Handlers/` and `Private/Utils/`; the only
  same-named hits are `FFoliageInfo::` (`FoliageHandler.cpp:468`, `:481`, `:498`, `:1228`, `:1232`)
  and `AActor::AddInstanceComponent` (six sites), neither of which touches an ISM instance array.
  Verified both halves of the parent ticket's scoping: `F-ism-per-instance-transforms` cites the
  absent `AddInstance` / `RemoveInstance` call sites in its diagnosis and its "Proposed verb shape —
  three verbs, not one" section proposes only `actor.get_instances`,
  `actor.set_instance_transforms` and `spatial.ground_instances`, all three of which its
  `#2-three-verbs-shipped` confirms landed in the proposed shape. Escape hatch measured against both
  the handler and the wiki: `actor.add_component` creates an empty HISM
  (`ComponentHandler.cpp:75-77`, `:87-88`) and — contrary to its own parameter text at `:47` and to
  `Saved/PinWright/wiki/actor.add_component.md` — **does** set `meshPath` on it, because `:113` casts
  to `UStaticMeshComponent` rather than testing the class name, and HISM derives from it. That doc
  error is noted for same-commit correction. The instance array itself stays unreachable through any
  typed verb; `object.call_function` reaches the five BlueprintCallable engine entry points
  (`InstancedStaticMeshComponent.h:271`, `:275`, `:417`, `:421`, `:431`) on the live component and is
  not blocked by the CDO refusal at `ObjectCallFunctionHandler.cpp:107-110`, which is what holds this
  at Medium rather than High.
