---
id: F-ism-create-and-clear-scatter
title: "Three typed verbs read, write and ground the instances of an ISM/HISM scatter and no verb creates or empties one — AddInstance / RemoveInstance / ClearInstances have zero call sites in the plugin, so the only way to populate the array the other three operate on is object.call_function per instance"
status: OPEN
severity: Medium
category: feature
tags: [actor, spatial, ism, hism, instanced-static-mesh, scatter, missing-verb, create, clear, add-instance, remove-instance, lifecycle-gap, data-loss, atomicity, python-execute, premise-corrected, vegetation]
encounters: 2
lastSeen: 2026-08-29T18:20:00+05:00
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
- `#2-hand-rolled-clear-then-refill-destroyed-50-instances` `OPEN` reporter — Second encounter,
  `encounters` 1 -> 2, and the first with a **data-loss** outcome. Re-speciating zone F of
  `PW_VegetationTest` (`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 1): a hand-rolled
  clear-then-refill helper **destroyed `HISM_ZF_Oak`'s 50 instances**. `clear_instances()` ran, then
  the refill raised `TypeError` and never wrote. Recovery was possible only because
  `dev/polish/p_zf_rework.py`'s `thin()` is a pure function of position and `dev/polish/out/zonef.json`
  still held the 183 pre-thin transforms; the replay reproduced n=50 with X/Y bounds identical to a
  census taken minutes earlier. **A caller without a replayable pure function loses them outright.**

  TWO PREMISES FROM THE FIELD REPORT DID NOT SURVIVE RE-DERIVATION, and both corrections matter to a
  fixer. (a) It was reported as *"`should_return_indices` is required POSITIONALLY"*. It is not
  positional — it is **required**. `PyGenUtil::ParseMethodParameters` tries the keyword dict for
  every parameter before any positional fallback (`PyGenUtil.cpp:1094-1097`, tuple fallback
  `:1112-1115`, and `:1104` errors when one argument arrives both ways), so a keyword call is legal
  for every param. What a `CPP_Default_<Param>` metadata entry controls is required-vs-optional, not
  keyword-vs-positional: `:999` builds that key, `:1007` stores it as `ParamDefaultValue`, and `:1117`
  admits the argument only if it was parsed **or** a default exists — otherwise `:1123` raises
  `"%s() required argument '%s' (pos %d) not found"`. `UInstancedStaticMeshComponent::AddInstances`
  (`C:/UE_5.8/.../Components/InstancedStaticMeshComponent.h:275`) declares
  `bShouldReturnIndices` with no C++ default while `bWorldSpace` and `bUpdateNavigation` have one, so
  UHT emits no `CPP_Default_bShouldReturnIndices` and that one argument is mandatory. (b) It was
  assumed PinWright taught the broken call. **It does not.** The shipped page passes the argument:
  `Saved/PinWright/wiki/level-building.instancing-and-scatter.md:20-21` (source
  `Plugins/PinWright/Docs/wiki-src/level-building.instancing-and-scatter.md:18-19`) reads
  `comp.add_instances(instance_transforms=transforms, should_return_indices=False, world_space=False,
  update_navigation=False)` — correct on UE 5.8. So this is **not** a wiki defect and is deliberately
  not filed as one; the page's only soft spot is `:25`, which frames the raise as a free retry
  (*"if the keyword form raises, use `add_instance(t, False)` per transform"*) without saying the
  retry can arrive after a destructive step.

  WHY IT LANDS ON THIS TICKET. The destructive sequence exists only because there is no typed
  clear/refill: the caller had to author `clear` and `add` as two statements inside one
  `python.execute` function, where a raise between them is unrecoverable. This ticket's own
  `## Proposed verb shape` already fixes it by construction — `actor.add_instances` routing through
  `AddInstances(..., bShouldReturnIndices=true, ...)` puts the argument C++-side where it cannot be
  omitted, and a server-side `remove_instances {all:true}` makes clear-then-refill one round trip
  with one owner of the failure. The evidence also sharpens the ticket's own third fixer point: it
  already reasons that the no-`FScopedTransaction` decision transfers to add but **not** to clear
  because *"nothing the response can echo reconstructs a discarded scatter cheaply"*. This is that
  case, observed — a discarded scatter recovered only by luck.

  THE DOCUMENTED WORKAROUND IS SAFE ON THIS AXIS, CHECKED RATHER THAN ASSUMED. `object.call_function`
  resolves defaults through the *same* `CPP_Default_<ParamName>` probe
  (`Handlers/Reflection/ObjectCallFunctionHandler.cpp:240-247`), so `bShouldReturnIndices` is
  required there too — but it refuses with `MISSING_PARAM` **before touching the object** (`:245-247`
  breaks the loop, `DestroyAll()` at `:260`, error at `:261-262`), and clear and add are two separate
  RPCs, so a failure leaves the caller still holding the transforms. **The destroy-then-fail hazard
  is specific to the `python.execute` route** — which the transcription ceiling forces at this scale:
  `Docs/map/vegetation-polish.md` § 5.7 records 5888 instances placed through `python.execute`
  precisely because there is no server-side handoff between `spatial.scatter_layout` and the instance
  writers. So the workaround `#1` relies on is safe per call and unusable per scatter, and the two
  facts are the same fact.

  CONTRAST WORTH KEEPING: PinWright's own ISM writer already has the atomicity the hand-rolled route
  cannot have. `actor.set_instance_transforms` refuses whole rather than in part on an `expectedCount`
  mismatch — `InstancedMeshHandler.cpp:377-390`, message *"NOTHING WAS WRITTEN. Instance indices are
  positional, so a component whose count changed has renumbered them"* — and again on a stale index
  (`:424-431`, *"refused whole rather than applied in part"*). That guard describes exactly the state
  a half-completed clear-then-refill leaves behind, on the one verb that already exists to be safe
  from it.

  SEVERITY UNCHANGED AT **Medium**, and the reasons the data loss does not move it are stated so a
  reviewer can disagree deliberately. Critical is *"a write that corrupts or loses asset data"*, and
  **no PinWright verb performed a write here**: the destroying call was the caller's
  `clear_instances()` inside `python.execute`, and the failing call was an engine binding raising
  correctly on a required argument. Rating a *missing-verb* ticket Critical for what a caller's
  hand-rolled substitute did would rate the absence of a verb by the worst thing anyone builds in its
  place, and every missing-verb ticket on this board would inherit that. What the encounter does
  change is the strength of `#1`'s Medium argument rather than the band: `#1` held Medium partly
  because *"`ClearInstances` in particular is a single no-argument call"*, which is true and beside
  the point — the gap is not that clear is hard, it is that clear and refill are not one operation
  and the plugin offers neither end. Reach declined again in both directions on `#1`'s reasoning,
  unchanged; and `encounters` is a same-severity work-ordering tiebreak, never a severity input.

  ON THE SESSION'S RECURRING CLASS, because this is its **complement and not a member**. The class
  stated on `B-foliage-paint-does-no-ground-projection` § *Same shape as* is *the call succeeds,
  every number it reports is correct, and the output is wrong because the deciding number was never
  reported.* This is the mirror: the call **fails, loudly and correctly**, and the output is still
  wrong because the failure arrived after the destructive step. Same session, same workflow, opposite
  failure geometry — and the same remedy, which is to put the pair behind one verb that can be
  atomic. The class member from this pass is
  `B-component-mesh-swap-silently-unseats-instances` (OPEN, Medium), which is the cost of the
  re-point callers reach for *instead of* clear-and-refill; the two are the two exits from the same
  missing capability and should be read together.
