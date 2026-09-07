---
id: F-foliage-remove-type-registration
title: "No verb removes a foliage type's REGISTRATION from a level — `foliage.remove` empties `FFoliageInfo::Instances` and leaves the `FoliageInfos` entry standing, and that entry is a hard reference the IFA hands the GC collector by hand, so an emptied type is undeletable (`asset.delete` -> ASSET_IN_USE) and unlistable at the same time (`foliage.get_instances` emits rows per instance and nothing per type), which is why this level carries 20 `/Game/Foliage/Auto_*` types plus 2 `PWScratch_*` as permanent residue"
status: OPEN
severity: Medium
category: feature
tags: [foliage, remove, foliage-type, instanced-foliage-actor, missing-verb, lifecycle-gap, asset-delete, asset-in-use, residue, level-hygiene, readback-blind-spot, python-execute, premise-corrected, source-only, vegetation]
encounters: 1
costly: 1
lastSeen: 2026-08-30T16:10:03+03:00
---

# The removal verb removes instances. The thing it removes them *from* is permanent.

`foliage.remove` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1131`)
is registered as *"Remove foliage instances by type or all"* and does exactly that. Both branches —
the `removeAll` walk at `:1215-1218` and the scoped `FindInfo` branch at `:1219-1224` — call the
same helper, `RemoveAllFoliageInstances` (`:1106-1127`), whose one write is
`Info.RemoveInstances(AllIndices, /*RebuildFoliageTree*/ true)` at `:1126`. The response
(`:1226-1237`) is `success` / `instancesRemoved` / `mode` / `foliageActorPath` / `existsAfter`.

Nothing in that path touches `AInstancedFoliageActor::FoliageInfos`. The type stays **registered**,
holding zero instances, for the life of the level. `RemoveFoliageType` — the engine call that would
remove it — appears **nowhere** under `Plugins/PinWright/Source/`, and the foliage namespace's whole
verb list at HEAD is six entries, all in this one file: `paint` (`:547`), `remove` (`:1131`),
`get_instances` (`:1244`), `add_type` (`:1486`), `add_instances` (`:1638`), `create_procedural`
(`:1935`). Create and populate ship; retire does not.

## The engine call exists, is linked, and is *not* out of reach — the premise, corrected

`AInstancedFoliageActor::RemoveFoliageType` is declared
`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/InstancedFoliageActor.h:241`, `FOLIAGE_API`, under
the comment at `:240`: *"Remove the FoliageType from the list, and all its instances."* Its body
(`Private/InstancedFoliage.cpp:3911-3933`) brackets the removal in
`UnregisterAllComponents()` (`:3914`) / `RegisterAllComponents()` (`:3932`), calls
`Info->Uninitialize()` (`:3925`) and then `FoliageInfos.Remove(FoliageType)` (`:3928`).

**It carries no `UFUNCTION`** — the header holds exactly two, both `BlueprintCallable` statics
inside `#if WITH_EDITOR` (`#endif` at `:290`): `AddInstances` (`:285-286`) and `RemoveAllInstances`
(`:288-289`).

**But "unreflected, therefore unreachable" is false, and the correction is the whole severity
argument.** `AInstancedFoliageActor::RemoveAllInstances` — the reflected one, exposed to Python as
`unreal.InstancedFoliageActor.remove_all_instances(world_context, foliage_type)` — has this body
(`InstancedFoliage.cpp:4986-4997`): iterate every `AInstancedFoliageActor` in the world and call
`IFA->RemoveFoliageType(&InFoliageType, 1)` at `:4994`. So the reflected surface reaches the
unreflected function, and `python.execute` **can** retire a type today. The honest statement of this
gap is therefore narrower than "it cannot be done": **no typed verb does it**, and the Python route
is a source dive behind a function whose name says *instances* while its body removes the *type*.

Linkage is already in place: `Plugins/PinWright/Source/PinWright/PinWright.Build.cs:69` lists
`"Foliage", "FoliageEdit"`, so a handler can call `RemoveFoliageType` directly. The fix is a call,
not a module change.

## Why the residue is not inert: undeletable and unlistable at the same time

**1. Undeletable.** `FoliageInfos` is a private, non-`UPROPERTY`
`TMap<TObjectPtr<UFoliageType>, TUniqueObj<FFoliageInfo>>` (`InstancedFoliageActor.h:43`) — which is
precisely why the class declares a manual `AddReferencedObjects` (`:59`). That function
(`InstancedFoliage.cpp:5208-5221`) walks the map and does
`Collector.AddReferencedObject(Pair.Key, This)` at `:5214`: **the map key is a hard, GC-tracked
reference to the `UFoliageType`, held by the level's IFA.** While the registration stands, the IFA
is a live in-memory referencer of the asset.

`asset.delete` then refuses, by design and correctly: `AssetDeletePolicy::DeleteAsset` calls
`ObjectTools::DeleteObjects` (`Private/Utils/AssetDeletePolicy.cpp:179`), and a `NumDeleted` of 0
sets `Result.bRefused = true` (`:190`) and gathers the holders (`:191-193`), which
`AssetManageHandler.cpp:839-847` emits as `refused:true` with
`errorCode: ASSET_IN_USE` (`:845`), repeated at the top level (`:942`). Nothing here is a defect —
the gate is behaving exactly as its own parameter doc promises (`AssetManageHandler.cpp:664`). The
defect is that the surface offers no way to *release* the reference the gate is protecting.

**2. Unlistable.** `foliage.get_instances` (`FoliageHandler.cpp:1244`) emits one row per **instance**
and nothing per registered type: the filtered branch walks `Info->Instances` off `FindInfo`
(`:1406`), the unfiltered branch walks `ForEachFoliageInfo` (`:1431`) emitting a row per instance,
and `count` is `InstancesArray.Num()` (`:1453-1454`). `orphanedInstanceCount` (`:1438`, `:1458`)
counts orphaned *instances*, not orphaned registrations. **A registration holding zero instances
contributes nothing to any field of any response.** So after a `foliage.remove` the type is at once
undeletable and invisible: the caller cannot remove it, and cannot see that it is there.

## The residue, counted

Measured on disk, not through the editor: `X:/src/unreal/EAContentExamples58/Content/Foliage/` holds
**20** `Auto_*.uasset` foliage types, plus `PWScratch_PaintRot.uasset` and
`PWScratch_VerifyRemove.uasset` — both untracked in git, i.e. scratch types minted during this
session's authoring passes. The `Auto_*` names are not misuse: they are what
`foliage.paint` / `foliage.add_instances` mint automatically whenever they are handed a bare
static-mesh path (`B-add-instances-auto-foliage-type-name-mismatch`,
`B-foliage-auto-type-no-disk-write`). Registration accrues on the ordinary authoring path; only the
retire half is missing.

**Stated as unverified, because it is the very thing no verb can answer:** how many of those 22 are
still registered in `/Game/Maps/PW_VegetationTest`'s IFA is not knowable from the RPC surface — see
§ 2 above. The disk count is a floor on how many types this workflow produced, not a measurement of
the level's registry.

## What DOES exist, so the ask is stated honestly

Three routes exist. None is a removal verb, and the second is worse than the residue.

- **`python.execute` -> `unreal.InstancedFoliageActor.remove_all_instances(world, type)`.** Real,
  one call, and it removes the registration (mechanism above). Its trap, which any doc note must
  carry: the engine function iterates **every** `AInstancedFoliageActor` in the world
  (`InstancedFoliage.cpp:4991-4995`), so on a shared level it also discards other callers' instances
  of that type — the same "scope is a property of the level rather than of the call" hazard as
  `B-ground-instances-default-component-foreign-scatter` (IN-REVIEW, Critical).
- **`asset.delete {force:true}`, then let the engine repair the level.** Force-delete nulls the
  key in place; `AInstancedFoliageActor::CleanupDeletedFoliageType()` (`InstancedFoliageActor.h:108`,
  body `InstancedFoliage.cpp:5180`) then prunes null keys, driven in a stock editor off
  `AssetRegistry.OnAssetRemoved` by `FFoliageEditModule::NotifyAssetRemoved`
  (`Editor/FoliageEdit/Private/FoliageEditModule.cpp:140-146`, bound at `:156`).
  `FoliageHandler.cpp:1417-1427` already reasons about exactly this state. **This is not a
  workaround.** `asset.delete`'s own `force` doc (`AssetManageHandler.cpp:670`) says force
  *"replaces EVERY in-memory pointer to the asset with null editor-wide and marks each of those
  packages dirty ... irreversibly (no transaction, no undo)"* and *"do not save any package named
  there, reload it with asset.reload"*. The package named here is the **map**, so the prescribed
  recovery is to discard the level's unsaved state. See `B-force-delete-nulls-referencers` and
  `B-asset-delete-force-delete-leaves-uasset-on-disk`.
- **Leave it.** The status quo, and what the 22 assets above record.

## Proposed verb shape

**`foliage.remove_type {foliageTypePath, requireEmpty?}`** ->
`{removed, instancesDiscarded, typesRemaining, foliageActorPath, existsAfter}`.

- Route through `IFA->RemoveFoliageType(&Type, 1)` on **this level's** IFA, deliberately *not*
  through the engine's `RemoveAllInstances`, so the scope is the actor the call resolved rather than
  every IFA in the world. That single choice is what separates the verb from its own workaround.
- **`requireEmpty` defaults to `true`.** `RemoveFoliageType`'s own comment
  (`InstancedFoliageActor.h:240`) says it removes *"all its instances"*, so an unguarded cleanup verb
  is a wholesale wipe wearing a hygiene verb's name. Refuse with a typed error naming the live
  count; `requireEmpty:false` is the deliberate wipe and must echo `instancesDiscarded`. The
  precedent for refusing rather than guessing a scope is this same file's
  `INVALID_ARGUMENT` / `ASSET_NOT_FOUND` gates on `foliage.remove` (`:1178-1192`,
  `E-foliage-remove-silent-edge-inputs`).
- **`typesRemaining` is the half that closes the readback hole**, and it is the cheaper half: a list
  of the paths the IFA still registers (with each one's instance count), from one
  `ForEachFoliageInfo` walk the file already performs at `:1215`, `:1431` and `:2367`. Without it a
  caller still cannot find residue to remove. If a fixer must ship only one piece, ship this one —
  it is also the field that would let anyone audit the claim in § The residue above.

**Folding it into `foliage.remove` as a third scope (`removeType:true`) is argued against, not
overlooked.** That verb's scope vocabulary is already contested
(`E-foliage-remove-mode-is-output-only`, OPEN, Low — the `mode` field is emitted and rejected as
input), and it is one commit past a Critical data-loss fix
(`B-foliage-remove-empties-ledger-not-component`, DONE). Widening its scope surface now is the wrong
moment; a sibling verb keeps the two blast radii separate.

## Not a duplicate of

- **`B-foliage-remove-empties-ledger-not-component`** (DONE, Critical) — the same verb's
  *instance-side* defect, now fixed; `RemoveAllFoliageInstances` (`FoliageHandler.cpp:1106-1127`) is
  that fix and is the code this ticket read. Instances vs registration are disjoint objects with
  disjoint lifetimes: that ticket made the removal reach the component, this one observes that the
  registration was never in scope for either the old code or the new.
- **`E-foliage-remove-mode-is-output-only`** (OPEN, Low) — same verb, input/output naming. Quoted
  above as the argument against folding this in.
- **`F-ism-create-and-clear-scatter`** (OPEN, Medium) — the **neighbour, not the duplicate**, and the
  distinction is the object: that ticket is about a plain ISM/HISM *scatter*, whose lifetime is its
  component's; this is about a *foliage type registration* on `AInstancedFoliageActor`, which
  outlives every instance it ever held and is keyed by an asset. Different handler, different engine
  API, and — the load-bearing difference — clearing a scatter leaves nothing behind, while emptying a
  foliage type leaves the thing this ticket is about.
- **`F-resimulate-existing-foliage-volume`** (OPEN, Medium) — different missing verb in the same
  namespace; quoted below as the severity precedent.
- **`F-foliage-get-procedural-types`** (OPEN, Medium) — reads which types a
  `UProceduralFoliageSpawner` *drives*. Adjacent to `typesRemaining` above but a different owner: a
  spawner's configured array vs the level actor's live registry.
- **`B-foliage-auto-type-no-disk-write`** (OPEN, High) and
  **`B-add-instances-auto-foliage-type-name-mismatch`** (IN-REVIEW, High) — the *source* of the
  residue, not its removal. Neither asks for a retire path.
- **`E-foliage-add-type-auto-save-undocumented`** (WONTFIX, Low) — the create side's persistence.

**The session's recurring class does not apply here, and forcing it would be a stretch.** The
canonical form (`B-foliage-paint-does-no-ground-projection` § *Same shape as*) is a call that
succeeds with every number correct and an output that is wrong because the deciding number went
unreported. `foliage.remove` reports nothing false: `instancesRemoved` is exact and `mode:"type"` is
a true statement about the scope that was applied. The nearest thing to the class is that a caller
can read `mode:"type"` as "the type is gone" — but that field is documented as scope
(`Saved/PinWright/wiki/foliage.remove.md`, 13:32 build) and the misreading is the caller's, not the
verb's. This is a missing verb, filed as one.

## Severity

**Medium, argued.** Impact class is the rubric's *"High or Medium: hard blocker with no workaround
(a stub, a missing verb, or rejecting valid input)"* — a missing verb, with a second-order
consequence (`ASSET_IN_USE` on a delete the caller cannot unblock through the surface).

It lands on **Medium and not High** for one reason, stated so a reviewer can disagree with it: the
workaround is real rather than theoretical. `python.execute` reaches a reflected `BlueprintCallable`
that removes the registration in a single call, which is the rubric's Medium verbatim — *"Doable,
but only via a documented workaround, a source dive, or many extra calls"* — and it is a source dive
on all three counts (absent from the wiki, findable only by reading the engine body, and named for
the wrong operation). It is explicitly **not** High: nothing is silently wrong anywhere here.
`foliage.remove` never claims to have removed a type, and `asset.delete`'s refusal is loud, typed
and correctly coded.

**The board has precedent both ways, and this follows the nearer one.** Counting `python.execute` as
a workaround: `F-ism-create-and-clear-scatter` (Medium — *"the workaround is real rather than
theoretical"*) and `F-resimulate-existing-foliage-volume` (Medium — *"`python.execute` reaches
`unreal.ProceduralFoliageEditorLibrary` directly, so a caller is not absolutely stuck ... this is the
source-dive case"*). Declining to count it: `F-niagara-remove-emitter` (DONE, Critical), whose
`## Workaround` section opens *"None inside MCP today"* and then names `python.execute` calling
`UNiagaraSystem::RemoveEmitterHandlesById`, dismissed as *"bypasses the typed-RPC surface and the
read-first workflow"*. I follow the foliage/ISM pair over the Niagara reading for two reasons: the
escape here is one reflected `BlueprintCallable`, not a native `NIAGARA_API` symbol poked through
reflection internals; and rating this above the two Mediums it sits beside in the same namespace,
with the same shape, would mis-order a picker that works the top band first.

**Reach modifier declined in both directions, and named.** No bump up: foliage-type retirement is not
an every-session path — a session can author for hours without needing one. No bump down either,
because the bump-down band is a *rare edge path* and this is not one: every `foliage.paint` /
`foliage.add_instances` given a bare mesh path mints a type, so registrations accrue on the ordinary
authoring path and the count on this level (22 assets, § The residue) is what one project's normal
use produced. Medium stands unmodified.

severity rationale: impact=missing verb (hard blocker band) argued down one step because a real
single-call `python.execute` route exists, so the Medium clause "doable only via a source dive"
× reach=neither every-session nor a rare edge path, declined in both directions -> Medium

## History
- `#1-no-verb-retires-a-foliage-type` `OPEN` reporter — Filed from a foliage-authoring hygiene
  question on `/Game/Maps/PW_VegetationTest`: scratch types written during authoring passes cannot be
  cleaned up. **Everything below is a source-only derivation at plugin HEAD `1a9e5778` against UE
  5.8; the running editor is the 13:32 build (`d8f1bc32`) and nothing here was executed against it.**
  MECHANISM: `foliage.remove` (`FoliageHandler.cpp:1131`) routes both branches (`:1215-1218`,
  `:1219-1224`) into `RemoveAllFoliageInstances` (`:1106-1127`), whose only write is
  `Info.RemoveInstances(...)` at `:1126`; `AInstancedFoliageActor::FoliageInfos` is never touched, so
  the registration survives with zero instances. `RemoveFoliageType` has **zero** occurrences under
  `Plugins/PinWright/Source/`, and the namespace's six verbs (`:547`, `:1131`, `:1244`, `:1486`,
  `:1638`, `:1935`) contain no retire path. **PREMISE CORRECTED, and it moved the severity.** The
  finding arrived as *"`RemoveFoliageType` is unreflected, so the type cannot be removed"*. Half of
  that is right: the function is `FOLIAGE_API` with no `UFUNCTION`
  (`InstancedFoliageActor.h:241`; the header's only two `UFUNCTION`s are `AddInstances` `:285-286`
  and `RemoveAllInstances` `:288-289`). The conclusion is wrong:
  `AInstancedFoliageActor::RemoveAllInstances` — reflected, `BlueprintCallable`,
  `unreal.InstancedFoliageActor.remove_all_instances` in Python — iterates every IFA in the world and
  calls `RemoveFoliageType` at `InstancedFoliage.cpp:4994`. So `python.execute` removes the
  registration today, in one call, and this is Medium rather than High because of it. (This is the
  same class of error as the earlier dossier claim that `FFoliageInfo::RemoveInstances` is never
  called in the plugin, which is false — it is called at `FoliageHandler.cpp:936`, `:953` and
  `:1126`. Both were re-derived rather than relayed.) CONSEQUENCES, both derived: **undeletable** —
  `FoliageInfos` is a non-`UPROPERTY` map (`InstancedFoliageActor.h:43`) whose keys are handed to the
  collector by the class's manual `AddReferencedObjects` (`InstancedFoliage.cpp:5214`), so the IFA is
  a live in-memory referencer; `asset.delete` therefore takes the refusal path
  `AssetDeletePolicy.cpp:179` -> `:190-193` -> `AssetManageHandler.cpp:845` (`ASSET_IN_USE`), which
  is the gate working correctly with no way to release what it protects. **Unlistable** —
  `foliage.get_instances` (`:1244`) emits rows per instance only (`:1406`, `:1431`, `:1453-1454`);
  `orphanedInstanceCount` (`:1438`, `:1458`) counts orphaned instances, not registrations, so a
  zero-instance type contributes to no field of any response and cannot be found at all. RESIDUE,
  measured on disk (not through the editor): `Content/Foliage/` holds 20 `Auto_*.uasset` types plus
  the untracked `PWScratch_PaintRot` / `PWScratch_VerifyRemove` from this session; how many are still
  registered in the level's IFA is explicitly **unverified**, because that is the question §
  Unlistable says cannot be asked. TWO OTHER ROUTES NAMED so the ask is honest: `asset.delete
  {force:true}` plus the engine's own `CleanupDeletedFoliageType` repair
  (`InstancedFoliageActor.h:108`, `InstancedFoliage.cpp:5180`, driven by
  `FoliageEditModule.cpp:140-146`) works but nulls references editor-wide and dirties the **map**,
  whose prescribed recovery per `AssetManageHandler.cpp:670` is to discard the level's unsaved state
  — worse than the residue; and the `python.execute` route above removes the type from every IFA in
  the world, inheriting `B-ground-instances-default-component-foreign-scatter`'s shared-scope hazard.
  DEDUP: grepped the board for `RemoveFoliageType` (**zero** hits), `ASSET_IN_USE` (four files, none
  about foliage types), `InstancedFoliageActor` (19 files), "foliage type" (19 files), and read
  `E-foliage-remove-mode-is-output-only`, `B-foliage-remove-empties-ledger-not-component`,
  `F-ism-create-and-clear-scatter`, `F-resimulate-existing-foliage-volume`,
  `E-foliage-add-type-auto-save-undocumented`, `B-foliage-auto-type-no-disk-write`,
  `B-add-instances-auto-foliage-type-name-mismatch` and `F-foliage-get-procedural-types` in full.
  Nothing owns the registration lifetime. The closest, `F-ism-create-and-clear-scatter`, is a
  neighbour on a different object: clearing an ISM scatter leaves nothing behind, emptying a foliage
  type leaves the registration this ticket is about. ASK: `foliage.remove_type {foliageTypePath,
  requireEmpty=true}` scoped to the resolved IFA (not the world), plus `typesRemaining` — the readback
  half, from a walk the file already does at `:1215` / `:1431` / `:2367`, and the piece to ship first
  if only one ships. Rated **Medium**: missing-verb impact class argued down one step because the
  `python.execute` route is real, following `F-ism-create-and-clear-scatter` and
  `F-resimulate-existing-foliage-volume`; the opposite precedent (`F-niagara-remove-emitter`, DONE, Critical,
  *"None inside MCP today"*) is named and declined, because that escape is a native API symbol and
  this one is a reflected `BlueprintCallable`. Reach declined both ways: not every-session, but not a
  rare edge path either — every bare-mesh-path paint mints a type, so registrations accrue on the
  normal authoring path.
