---
id: B-wiki-hism-recipe-resets-actor-transform
title: "The wiki's HISM recipe tells callers to set root_component by reflection on a spawned actor, which orphans the DefaultSceneRoot that was carrying the spawn location — so the layer builds at the world origin instead of where it was authored, and spatial.ground_instances then seats it there and reports placed: 41"
status: OPEN
severity: Medium
category: bug
tags: [wiki, wiki-src, docs, level-building, instancing-and-scatter, hism, ism, python-execute, root-component, transform, world-origin, silent-wrong-output, shipped-artefact]
encounters: 2
costly: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# A documented write recipe that silently relocates everything it builds

`Docs/wiki-src/level-building.instancing-and-scatter.md:12-21` is the plugin's prescribed route for
building an instanced layer you own. Line `:16`, verbatim, comment included:

```python
a.set_editor_property('root_component', comp)          # without this the component is not the root
```

and `:43` doubles down on it in prose:

> **`register_component()` and `is_registered()` do not exist in Python either — same family, same
> cause.** … Do not write them and do not write a registration read-back around them. **The recipe
> above needs neither: `set_editor_property('root_component', comp)` on a spawned actor is what puts
> the component in the level**, and `get_instance_count()` is the read-back that proves the write
> landed. If you need a component that is registered by C++ at creation time, use
> `call("actor.add_component")` instead of building it from Python.

Both lines ship. The generated page carries them at
`Saved/PinWright/wiki/level-building.instancing-and-scatter.md:18` and `:45` — the generator prepends
a two-line banner, and is otherwise byte-identical through this section. `Docs/wiki-src/` is the
source and `Saved/PinWright/wiki/` is regenerated at editor launch, so **the fix goes in `wiki-src`**
and the shipped page follows on the next launch.

**Measured, live, and expensively.** Following the recipe on a real layer, the instances authored
relative to the actor's spawn point landed roughly **17 km away, at the world origin**.
`spatial.ground_instances` (`Handlers/Spatial/GroundPlacementHandler.cpp:1235`) was then run on the
result and dutifully seated 41 of them onto whatever was under the origin, reporting `placed: 41`
(`:1555`). Every number in that response is correct. The layer is in the wrong place.

## Why it happens — and it is not a "reset"

There is no transform to reset. `AActor` stores no location of its own:

```cpp
/** Returns the location of the RootComponent of this Actor*/
inline FVector GetActorLocation() const
{
    return TemplateGetActorLocation(ToRawPtr(RootComponent));
}
```

`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h:2501-2505`. The actor's
transform *is* its root component's transform, so replacing the root replaces the transform.

The spawn location lives on the component the recipe throws away.
`unreal.EditorActorSubsystem().spawn_actor_from_class(unreal.Actor, ...)` (page `:13`) routes
`UEditorActorSubsystem::SpawnActorFromClass`
(`Editor/UnrealEd/Private/Subsystems/EditorActorSubsystem.cpp:521-537`) into
`InternalActorUtilitiesSubsystemLibrary::SpawnActor` (`:99-150`), which places the actor through
`FLevelEditorViewportClient::TryPlacingActorFromObject` (`:140`) — i.e. through an **actor factory**.
For a bare `AActor` that factory is `UActorFactoryEmptyActor`
(`Editor/UnrealEd/Private/Factories/ActorFactory.cpp:1306-1330`), whose whole body is:

```cpp
NewActor = Super::SpawnActor(InAsset, InLevel, InTransform, InSpawnParams);

USceneComponent* RootComponent = NewObject<USceneComponent>(NewActor, USceneComponent::GetDefaultSceneRootVariableName(), RF_Transactional);
RootComponent->Mobility = EComponentMobility::Movable;
RootComponent->bVisualizeComponent = bVisualizeActor;
RootComponent->SetWorldTransform(InTransform);      // :1321  <- the spawn location lives HERE

NewActor->SetRootComponent(RootComponent);          // :1323
NewActor->AddInstanceComponent(RootComponent);      // :1324
RootComponent->RegisterComponent();                 // :1326
```

So the 17 km is stored on a `DefaultSceneRoot` created at `:1318` and placed at `:1321`. The recipe's
`comp` is a freshly `unreal.new_object`'d HISM (page `:15`) — never attached, never placed, relative
transform identity. Pointing `RootComponent` at it makes `GetActorLocation()` read identity, and the
old root is left registered and still in `InstanceComponents`, holding the only copy of the placement
nobody is reading any more.

The instances then follow: `comp.add_instances(..., world_space=False, ...)` (page `:18-19`) writes
them **relative** to a root that is now at the origin.

## A second defect in the same line, smaller but worth fixing together

`set_editor_property('root_component', comp)` writes the `UPROPERTY` directly. `RootComponent` is
declared with a getter and **no setter**:

```cpp
UPROPERTY(BlueprintGetter=K2_GetRootComponent, Category="Transformation")
TObjectPtr<USceneComponent> RootComponent;
```

`Actor.h:1022-1024`. So the reflection write bypasses `AActor::SetRootComponent`
(`Runtime/Engine/Private/Actor.cpp:5367-5391`) entirely, and with it `Modify()` (`:5374`) —
no undo record — and `NotifyIsRootComponentChanged` on both the new root (`:5382`) and the old one
(`:5387`). The page's own `call("actor.add_component")` alternative, named in the last sentence of
`:43`, is the route that gets all of this from C++ for free.

## Half of this is already owned and already fixed — say so, and notice where the fix landed

The companion symptom — that `register_component()` and `is_registered()` raise `AttributeError` on
`UActorComponent` in UE 5.8 Python — belongs to `B-python-execute-private-scope-leaks-sys-modules`
(IN-REVIEW, Medium) item (c) (`:116-120` of that ticket), and its `#1` records the remedy as landed:
*"added (c)'s missing pair — `register_component()` / `is_registered()` carry no UFUNCTION and
`AddComponentByClass` is `ScriptNoExport` on 5.8 — to
`Docs/wiki-src/level-building.instancing-and-scatter.md`."* **Verified present**: that is the
paragraph at `:43`. It is correct, it is not re-filed here, and nothing in this ticket asks for it to
change.

**The sharp part is that the fix for that half is in the same paragraph that endorses the broken
line.** The sentence immediately after the `AttributeError` warning is *"The recipe above needs
neither: `set_editor_property('root_component', comp)` on a spawned actor is what puts the component
in the level"* — a correct finding about two missing `UFUNCTION`s used to argue that the surviving
call is sufficient, in a paragraph added specifically to make the page more accurate. A reader who
trusts the paragraph because its first half is demonstrably right inherits its second half.

## What the page should say instead

Not prescribed here, because the replacement was not tested. What was determined:

- **Re-applying the transform after the assignment is sufficient for the placement.** Once `comp` is
  the root there *is* a root, so `a.set_actor_location(...)` / `set_actor_transform(...)` succeeds.
  A recipe that captures the intended location before the swap and re-applies it after is the
  minimal correction.
- **It is not sufficient for the rest of the line.** The undo record, both
  `NotifyIsRootComponentChanged` calls, the orphaned still-registered `DefaultSceneRoot`, and the
  question the page asserts without evidence — whether an unregistered `new_object`'d component
  becomes a real level component purely by being pointed at from `RootComponent` — are all
  unaddressed by a re-apply, and the last of those was **not** determined in this pass.
- So the ask is: **the recipe must be re-derived and re-tested on 5.8**, not patched. The two
  candidates to test are (a) keep the factory's `DefaultSceneRoot` as root and attach the HISM to it
  rather than replacing it, and (b) `call("actor.add_component")`, which the page already names in
  the same paragraph and which registers from C++. Whichever survives, the page should state the
  transform consequence explicitly, because a reader cannot derive it: `AActor` has no transform of
  its own, so swapping the root swaps the placement.
- **Add a read-back that would have caught it.** `get_instance_count()` — the page's current
  read-back at `:43` — counts instances and cannot see where they are. `a.get_actor_location()`
  after the swap is one line and is the assertion that fails.

## The open question, answered on one side: candidate (b) works, and the page's read-back cannot see registration

§ *What the page should say instead* left two candidates untested and named the undetermined
question: *"whether an unregistered `new_object`'d component becomes a real level component purely
by being pointed at from `RootComponent`"*. A later session (`Docs/map/vegetation-polish.md`
§ 5.4, look-dev polish over `PW_VegetationTest`, 2026-08-29) built 12 instance-level HISM layers
and measured three things that move it.

**Candidate (b) works, 12/12.** `call("actor.add_component", {actorName, componentType:
"HierarchicalInstancedStaticMeshComponent", meshPath})` succeeded on every one of twelve rootless
bare-`AActor` holders and produced a component that draws. Source confirms why it is the safe
route and not merely the working one: `Handlers/Actor/ComponentHandler.cpp:87-88` constructs it,
`:96-97` runs `AddInstanceComponent` + `OnComponentCreated`, `:157` calls `RegisterComponent()`
from C++ — and `:100-102` attaches to the root **only if the actor already has one**, so the verb
never writes `RootComponent` and is structurally incapable of the displacement this ticket is
about. `meshPath` also reaches a HISM despite its parameter description at `:47` saying it is
"only used when componentType is StaticMeshComponent": the assignment at `:113-119` casts to
`UStaticMeshComponent`, and HISM derives from ISM derives from that. The description is narrower
than the behaviour, which is worth a line on the same fix.

**The two obvious Python alternatives are closed, and one of them for a newly-cited reason.**
`AActor.add_instance_component` and `AActor.add_component_by_class` are both `AttributeError` from
bundled UE 5.8 Python. The page already records the `AddComponentByClass` half (`ScriptNoExport`);
the other half is `AActor::AddInstanceComponent`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h:4348`), a plain
`ENGINE_API void` carrying **no `UFUNCTION`** at all. So after `unreal.new_object` there is no
reflected call that puts the component anywhere — which is exactly the corner the recipe's
`set_editor_property('root_component', comp)` line was invented to escape.

**The read-back the page would reach for cannot answer the open question.** A component created by
`unreal.new_object(..., outer=actor)` and never registered is nevertheless enumerated by the
actor's component list. That is by construction, not by accident:
`UActorComponent::PostInitProperties`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Components/ActorComponent.cpp:581`) calls
`OwnerPrivate->AddOwnedComponent(this)` at `:599` for any component whose `CreationMethod` is not
`Instance` — the default for a bare `NewObject` — and `AActor::AddOwnedComponent`
(`Runtime/Engine/Private/Actor.cpp:3792`) inserts into `OwnedComponents` at `:3802`. The only
reflected enumerator, `K2_GetComponentsByClass` (`Actor.h:3806-3807`, Python
`get_components_by_class`), reads that set. **`OwnedComponents` membership is established at
construction; registration is a separate, later, unreflected step.** So the enumeration answers
*"does this actor own this object"* and is read as *"is this component in the level"*, and
`is_registered()` — the call that would tell them apart — is the one this page already documents
as unreachable from Python. Any re-derivation of the recipe therefore has to read a registration
signal from C++ or from an observable render effect; it cannot be settled from the Python side at
all. That is a constraint on the fix, not a new defect.

What this does **not** settle: whether pointing `RootComponent` at an unregistered component
registers it. The measurement above is of the state *before* any root assignment, and the polish
run took candidate (b) rather than repairing candidate (a). The question stays open; what changed
is that the cheapest way to answer it has been ruled out.

**Flagged for the re-derivation, source-only and NOT measured here:** a Python-built component is
in `OwnedComponents` (above) but nothing puts it in `AActor::InstanceComponents`, which is what
`actor.add_component` gets from `ComponentHandler.cpp:96`. Whether such a component survives a
save and reload on its own — as opposed to surviving only because `RootComponent` happens to hold
a serialized reference to it — was not tested and should be, since the replacement recipe's
persistence depends on it.

## Same shape as

`B-foliage-paint-does-no-ground-projection` carries the fullest statement of the class: *the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number was
never reported.* Here the deciding number is the actor's location, and the confirming green number —
`placed: 41` — comes from a different verb entirely, which is what makes it convincing.

`F-scatter-layout-verb` (DONE, Medium) — same page, adjacent section: it quotes the scatter doctrine
at `:74-79` and files the gap between doctrine and implementation. Same artefact, different failure
mode: that page section is *right and unimplemented*; this one is *implemented and wrong*.

`F-ism-per-instance-transforms` (IN-REVIEW, High) — the capability this recipe is a hand-rolled
substitute for. Its refusal text already tells callers a non-foliage ISM/HISM "has to be re-scattered
by whatever built it", and this page is what most callers use to build it, so a fixer landing that
verb should re-point this recipe at it rather than repairing the Python.

`B-python-execute-private-scope-leaks-sys-modules` (IN-REVIEW, Medium) — owns the `AttributeError`
half and owns the paragraph that has to change.

Wiki-accuracy family, closest three: `E-cloth-verbs-wiki-overpromise-create-section` (IN-REVIEW,
Medium), `E-level-structure-wiki-advertises-broken-save-path` (OPEN, Low),
`E-level-structure-wp-wiki-advertises-dead-end` (IN-REVIEW, Low). **All three are `E-`; this one is
`B-`, deliberately.** Those pages advertise a verb that does less than claimed — a caller who follows
them gets nothing and knows it. This page hands a caller a working-looking script that silently moves
their content, which is a defect in a shipped artefact rather than an overpromise about another one.

## Verification status

The symptom is RPC-verified and measured (17 km displacement, `placed: 41` from
`spatial.ground_instances`). The *mechanism* is source-read: the chain
`SpawnActorFromClass` -> `TryPlacingActorFromObject` -> `UActorFactoryEmptyActor::SpawnActor` ->
`SetWorldTransform` on the `DefaultSceneRoot`, plus `GetActorLocation()` reading through
`RootComponent`, was traced in engine source and not confirmed by inspecting the orphaned component
in a live editor. The one-line check that would confirm it: after the swap, read
`a.get_actor_location()` and the old `DefaultSceneRoot`'s world location — this ticket predicts
`(0,0,0)` and the intended spawn point respectively.

severity rationale: impact=one band below High, argued rather than asserted — the SHAPE is the High band's silent wrong data on a normal path (a documented recipe relocates authored content by 17 km, the caller has no signal, and a green `placed: 41` from an unrelated verb actively confirms the damage), and one band is deducted because the defective artefact is documentation rather than a verb: the caller's own `python.execute` script is the proximate writer, no PinWright verb returned a false field, and a single `set_actor_location` after the swap undoes the whole effect once the cause is known -> Medium. Explicitly NOT Low: the rubric's Low band is "docs, discoverability, naming" — friction where the doc fails to help — and this doc actively misdirects, costing a re-run of a whole scatter pass and, on a layer that was ground-seated before anyone noticed, a re-run of that too; a doc that produces wrong output is not the same object as a doc that produces no output, which is also why this is filed `B-` where the rest of the wiki-accuracy family is `E-`. Not High: nothing is corrupted or unrecoverable, the instances are intact and the actor is one call from correct. reach modifier DECLINED in both directions — `level-building.instancing-and-scatter` is one of ~40 wiki pages and this recipe is one of five sections on it, so it is neither an every-session path (no bump up) nor a rare edge (no bump down); the reading that would bump it up is that this is the ONLY documented route to a caller-owned HISM layer, which is true but is a statement about the page's importance, not about how often the method runs, and the reading that would bump it down is that it has been hit once, which is an `encounters` signal the rubric excludes from severity outright -> Medium

## History
- `#1-root-swap-orphans-the-placement` `OPEN` reporter — Symptom measured live on a real layer: following the wiki's HISM recipe, instances authored relative to the actor's spawn point built roughly 17 km away at the world origin, and `spatial.ground_instances` (`Handlers/Spatial/GroundPlacementHandler.cpp:1235`) then seated 41 of them there and reported `placed: 41` (`:1555`) — a correct number confirming a wrong result. Page citations re-derived, not inherited: `Docs/wiki-src/level-building.instancing-and-scatter.md:16` is `a.set_editor_property('root_component', comp)          # without this the component is not the root`, and `:43` doubles down in prose ("The recipe above needs neither: `set_editor_property('root_component', comp)` on a spawned actor is what puts the component in the level"). Both ship: `Saved/PinWright/wiki/level-building.instancing-and-scatter.md:18` and `:45`, the generated file differing only by a two-line banner; `wiki-src` is the source, so the fix goes there. MECHANISM (source-read, engine): it is not a reset — `AActor` has no transform of its own, `GetActorLocation()` is `TemplateGetActorLocation(RootComponent)` (`Runtime/Engine/Classes/GameFramework/Actor.h:2501-2505`), so replacing the root replaces the placement. The spawn location was stored on the component the recipe discards: `spawn_actor_from_class` (`Editor/UnrealEd/Private/Subsystems/EditorActorSubsystem.cpp:521-537`) reaches `TryPlacingActorFromObject` (`:140`) and thus `UActorFactoryEmptyActor::SpawnActor` (`Editor/UnrealEd/Private/Factories/ActorFactory.cpp:1311-1330`), which creates a `DefaultSceneRoot` (`:1318`), calls `RootComponent->SetWorldTransform(InTransform)` (`:1321`), then `SetRootComponent` (`:1323`), `AddInstanceComponent` (`:1324`) and `RegisterComponent()` (`:1326`). The recipe's `comp` is an unattached `new_object`'d HISM at identity, so after the swap the actor reads (0,0,0) and `add_instances(..., world_space=False, ...)` (page `:18-19`) writes every instance relative to it. Second, smaller defect in the same line: `RootComponent` carries `BlueprintGetter=K2_GetRootComponent` and no setter (`Actor.h:1022-1024`), so the reflection write bypasses `AActor::SetRootComponent` (`Runtime/Engine/Private/Actor.cpp:5367-5391`) and with it `Modify()` (`:5374`, no undo record) and `NotifyIsRootComponentChanged` on both components (`:5382`, `:5387`); the old root stays registered and in `InstanceComponents`. HALF OF THIS IS ALREADY OWNED AND FIXED, and is deliberately NOT re-filed: the `register_component()` / `is_registered()` `AttributeError` belongs to `B-python-execute-private-scope-leaks-sys-modules` (IN-REVIEW, Medium) item (c), whose `#1` records the doc paragraph as added — verified present, it is the paragraph at `:43`. That is also the sharpest thing here and is stated plainly in the body: the paragraph carrying that landed fix is the same paragraph that endorses the broken line, so a reader who trusts its correct first half inherits its wrong second half. REPLACEMENT NOT PRESCRIBED, deliberately: re-applying the transform after the swap is sufficient for the placement (there is a root by then, so `set_actor_location` succeeds) but not for the undo record, the notifications, the orphaned still-registered `DefaultSceneRoot`, or the page's untested assertion that pointing `RootComponent` at an unregistered component is what "puts it in the level" — that last was not determined in this pass. Framed as: the recipe must be re-derived and re-tested on 5.8, testing (a) attach-to-the-existing-DefaultSceneRoot and (b) `call("actor.add_component")`, which the same paragraph already names. Also asked for: replace the `get_instance_count()` read-back, which cannot see where instances are, with `a.get_actor_location()`, which is the one line that fails. Dedup: grepped the board for `root_component`, `SetRootComponent`, `instancing-and-scatter`, `HISM`, `HierarchicalInstancedStaticMesh` and every `*wiki*` filename. `root_component` matches only `B-effect-geometry-fixture-not-at-claimed-column` (a fixture's own coordinates, unrelated). `instancing-and-scatter` matches only `F-scatter-layout-verb` (DONE — same page, the scatter-doctrine section at `:74-79`, doctrine-without-implementation rather than a wrong recipe) and `B-python-execute-private-scope-leaks-sys-modules`. The ~20-strong wiki family is entirely `E-` and entirely about verbs that do less than the page claims; nothing on the board covers a wiki recipe that produces wrong output. `F-ism-per-instance-transforms` (IN-REVIEW, High) is the capability this recipe substitutes for and is the right place to re-point it if that verb lands first.
- `#2-candidate-b-works-and-ownedcomponents-is-not-registration` `OPEN` reporter — Encounter from
  a later session (`Docs/map/vegetation-polish.md` § 5.4, look-dev polish over
  `PW_VegetationTest`), which built 12 instance-level HISM layers and moved `#1`'s named open
  question on one side. **Candidate (b) verified working 12/12**: `actor.add_component
  {componentType:"HierarchicalInstancedStaticMeshComponent", meshPath}` on rootless bare-`AActor`
  holders, and source says why it is the safe route rather than merely a working one —
  `ComponentHandler.cpp:87-88` constructs, `:96-97` `AddInstanceComponent` + `OnComponentCreated`,
  `:157` `RegisterComponent()` from C++, and `:100-102` attaches to a root only when one already
  exists, so the verb never writes `RootComponent` and cannot reproduce this ticket's
  displacement. Incidental doc/behaviour mismatch found on the same verb: `meshPath`'s description
  at `:47` claims StaticMeshComponent-only while the assignment at `:113-119` casts to
  `UStaticMeshComponent` and therefore reaches ISM/HISM. **Both Python alternatives confirmed
  closed**, one with a citation this ticket did not have: `AActor::AddInstanceComponent`
  (`Actor.h:4348`) is a plain `ENGINE_API void` with no `UFUNCTION`, alongside the already-recorded
  `ScriptNoExport` on `AddComponentByClass`. **The load-bearing new fact is negative**: a
  `unreal.new_object(..., outer=actor)` component that was never registered is still enumerated by
  `get_components_by_class`, because `UActorComponent::PostInitProperties`
  (`ActorComponent.cpp:581`) calls `AddOwnedComponent` at `:599` for any non-`Instance`
  `CreationMethod` and `AActor::AddOwnedComponent` (`Actor.cpp:3792`) inserts at `:3802`, while the
  only reflected enumerator `K2_GetComponentsByClass` (`Actor.h:3806-3807`) reads that same set.
  `OwnedComponents` membership is a construction-time fact and registration is a separate
  unreflected step, so the enumeration answers "does this actor own this object" and reads as "is
  this component in the level" — the same shape as this ticket's `placed: 41`. Consequence for the
  fix, recorded as a constraint rather than a new defect: `#1`'s open question cannot be settled
  from Python at all, because `is_registered()` is unreachable there; it needs a C++ signal or an
  observable render effect. Explicitly NOT settled: whether pointing `RootComponent` at an
  unregistered component registers it — the polish run took candidate (b) instead of repairing
  (a), so the measurement is of the pre-assignment state only. One item flagged **source-only and
  untested** for whoever re-derives the recipe: a Python-built component reaches `OwnedComponents`
  but nothing puts it in `AActor::InstanceComponents` (which `ComponentHandler.cpp:96` supplies),
  so its survival across save/reload independent of the `RootComponent` reference is unverified.
  Status left `OPEN` — nothing here fixes the shipped page; severity left **Medium**, since the
  new evidence narrows the remedy rather than widening the impact. `encounters` absent → 2.
