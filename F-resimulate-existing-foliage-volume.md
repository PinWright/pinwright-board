---
id: F-resimulate-existing-foliage-volume
title: "The ProceduralFoliageEditorLibrary reflection path is already in the tree but is welded inside foliage.create_procedural's creation flow, so no verb re-simulates or clears an AProceduralFoliageVolume that already exists — and object.call_function cannot substitute, because the library's four entry points are statics and the verb refuses ClassDefaultObject targets"
status: OPEN
severity: Medium
category: feature
tags: [foliage, procedural-foliage, resimulate, clear, create_procedural, reflection, call_function, cdo-refusal, missing-verb, iteration-loop]
encounters: 2
lastSeen: 2026-08-29T18:00:00+05:00
---

# The hard part is done, and it only runs once, at birth

The reflection bridge into `FoliageEdit` is present and working:

    UClass *LibClass = FindObject<UClass>(
        nullptr, TEXT("/Script/FoliageEdit.ProceduralFoliageEditorLibrary"));
    UFunction *ResimFn = LibClass ? LibClass->FindFunctionByName(
                                        TEXT("ResimulateProceduralFoliageComponents"))
                                  : nullptr;

`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1549-1553`,
dispatched via `CDO->ProcessEvent(ResimFn, &Params)` at `:1566-1569`, with the parameter-layout
mirror struct at `:1557-1560`. It handles the two things that make this bridge hard — the class
carries no `*_API` export so it cannot be linked cross-module, and the real `AddInstances` callback
lives in `FoliageEdit`'s private `FoliageEdMode.h` — and the comment at `:1534-1548` documents both.

**It is welded into the creation flow.** The whole block sits inside
`if (UProceduralFoliageComponent *ProcComp = Volume->ProceduralComponent)` (`:1526`), where `Volume`
is the `AProceduralFoliageVolume` this same call spawned four statements earlier (`:1489-1491`).
There is no other caller: `Resimulate`, `ProceduralFoliageEditorLibrary`,
`AProceduralFoliageVolume` and `UProceduralFoliageComponent` appear in exactly one non-test file in
the entire `Plugins/PinWright/Source` tree, and every occurrence is inside
`foliage.create_procedural`'s body (`FoliageHandler.cpp:25`, `:1489-1490`, `:1524-1570`).

So the answer to *"does any verb re-simulate or clear an already-existing volume?"* is **no** — and
the reason is not a missing capability but a missing entry point.

## Three of the library's four functions are unreachable, and the fourth only at creation

`C:/UE_5.8/Engine/Source/Editor/FoliageEdit/Public/ProceduralFoliageEditorLibrary.h:29-38` declares
four `UFUNCTION(BlueprintCallable)` statics:

| function | reachable through PinWright? |
| --- | --- |
| `ResimulateProceduralFoliageComponents` | only inside `foliage.create_procedural`, on the volume it just spawned |
| `ResimulateProceduralFoliageVolumes` | no |
| `ClearProceduralFoliageComponents` | no |
| `ClearProceduralFoliageVolumes` | no |

## `object.call_function` cannot substitute, and this is what makes the gap hard rather than merely awkward

The usual escape hatch for a `BlueprintCallable` engine function is `object.call_function`. It is
closed here by construction, for two independent reasons:

1. **The library's functions are `static`.** Invoking a static reflected `UFUNCTION` means calling
   `ProcessEvent` on the class default object — which is exactly what the plugin's own code does at
   `FoliageHandler.cpp:1566`. But `object.call_function` refuses CDO targets outright:

       if (Target->HasAnyFlags(RF_ClassDefaultObject))
       {
           ... FString::Printf(TEXT("Refusing to invoke on ClassDefaultObject: %s"), *ObjectPath)

   `Handlers/Reflection/ObjectCallFunctionHandler.cpp:107-110`, and the refusal is advertised in the
   verb's own summary (`:67`, rendered to `Saved/PinWright/wiki/object.call_function.md`). That
   refusal is right in general and is not what this ticket asks to change.

2. **There is nothing on the instance side to call instead.**
   `C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/ProceduralFoliageComponent.h` declares **no
   `UFUNCTION` at all**, so the component itself exposes no reflected simulate/clear that
   `object.call_function` could reach on a live object.

The remaining route is `python.execute` into `unreal.ProceduralFoliageEditorLibrary`. That works,
and it is why this is Medium rather than higher — but it is a source dive, and it is the exact shape
of workaround this plugin exists to remove.

## Why the missing loop matters more than it looks

Procedural foliage is not a one-shot verb; it is a tuning loop. The species' `OverlapPriority`
ordering, `MaxInitialAge`, `ProceduralScale`, `InitialSeedDensity` and the tile grid all need
several passes before a scatter reads correctly (see
`F-procedural-foliage-simulation-knobs-unreachable` for what those passes cost). Today each pass
means calling `foliage.create_procedural` again, which:

- spawns **another** `AProceduralFoliageVolume` (`:1489-1491`) rather than re-running the one in the
  level;
- creates **another** `UProceduralFoliageSpawner` and a fresh `_FT_<n>` set under the hardcoded
  `/Game/ProceduralFoliage` (`:1358`, `:1361-1364`, `:1439-1441`), accumulating assets per
  iteration;
- and gives no way to remove the previous pass's instances first, because `Clear*` is unreachable —
  so the level acquires overlapping scatters rather than a corrected one.

That last point is the sharp one. **`foliage.remove` is not the fallback**: it empties
`FFoliageInfo::Instances` and never touches the HISM
(`B-foliage-remove-empties-ledger-not-component`, OPEN, **Critical**), so the previous pass keeps
rendering while both the mutator and its readback report it gone. The one correct way to clear a
procedural pass is `ClearProceduralFoliageComponents`, which is the function this ticket is about.

It also blocks the obvious workaround for the knobs ticket: a caller *can* `property.set`
`ProceduralScale` / `InitialSeedDensity` / `OverlapPriority` onto the generated `_FT_<n>` assets,
but nothing then re-runs the simulation to consume them, so the write is inert. The two tickets
close each other's escape hatches; either one alone leaves the other's workaround broken.

## The "inert write" boundary, measured: `Mesh` is the exception, and it is both a partial workaround and a trap

§ *Why the missing loop matters more than it looks* says a `property.set` of `ProceduralScale` /
`InitialSeedDensity` / `OverlapPriority` onto a generated `_FT_<n>` asset is inert until something
re-runs the simulation. **That is correct for the three properties it names and false as a general
rule**, and the exception is the one a species swap needs.

Measured (`Docs/map/vegetation-style-split.md` § Findings 5, zone C/D re-speciation over
`PW_VegetationTest`): changing `UFoliageType::Mesh` on an already-simulated type **re-points every
already-spawned instance on the next frame**. No resimulate, no level reload; a component census
afterwards reports the new mesh, at the instance transforms the old simulation produced.

Mechanism, and it explains the boundary rather than merely recording it. The three simulation
knobs are *inputs to a simulation that has already run*, so nothing consumes them again. `Mesh` is
not a simulation input at all — it is a render-time indirection, and the engine pushes it eagerly
on the property-change notification:

- `UFoliageType::PostEditChangeProperty`
  (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:1035` computes
  `bSourceChanged`; the gate `IsFoliageReallocationRequiredForPropertyChange` at `:1042` guards a
  loop over every loaded `AInstancedFoliageActor` (`:1044`) calling
  `NotifyFoliageTypeChanged(this, bSourceChanged)` at `:1048`), paired with `PreEditChange` →
  `NotifyFoliageTypeWillChange` at `:1060-1062`;
- through `FFoliageInfo::NotifyFoliageTypeChanged` (`:2185-2188`) into
  `FFoliageStaticMesh::NotifyFoliageTypeChanged` (`:1449`), which runs `CheckComponentClass`
  (`:1452`), `UpdateComponentSettings` (`:1453`) and, when the source changed,
  `Component->BuildTreeIfOutdated(true, false)` (`:1466`).

So the re-point is **notification-driven**: it follows any write that dispatches
`PostEditChangeProperty`, and a write that bypasses the notification would not get it — the same
distinction that governs package dirtying elsewhere on this board.

**Why it matters for this ticket, on both sides.**

*Partial workaround.* A species change — the commonest reason to want the tuning loop this ticket
asks for — is the one edit that does **not** need `foliage.resimulate`. It lands immediately and is
observable immediately, which makes it a cheap way to preview a swap before paying for a
re-simulate. It narrows this ticket's scope rather than closing it: distribution, density,
clustering and age all still require the simulation to re-run, and those are what the knobs
ticket is about.

*Trap, and a sharper one than it looks.* Anyone reasoning "instances are frozen against the type
asset until I re-simulate" is wrong, and one branch of the path above reaches
`FFoliageStaticMesh::Reapply`: `CheckComponentClass` destroys and rebuilds the component when the
new mesh implies a different component class, and calls `Reapply(InSettings)`
(`InstancedFoliage.cpp:1571-1584`). **`Reapply` is the reconciler
`B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) identifies as the thing that
converts that ticket's ledger/component divergence into real deletion** (`:1859-1867`). A mesh
swap is therefore a *trigger* for that latent loss — conditional on the component class actually
changing, which a plain mesh-for-mesh swap need not do, so this is a hazard to check for rather
than a demonstrated one. Whoever implements `foliage.resimulate`'s `clearFirst` should read that
ticket first regardless: it is already cited below as a warning about what a clear path must not
do, and this is a second route into the same reconciler.

## Proposed verb shape

**`foliage.resimulate`** — `{actorName, clearFirst?}` → `{resimulated, cleared, instancesBefore,
instancesAfter, instancesDelta}`.

- `actorName` resolves an existing `AProceduralFoliageVolume` (or any actor carrying a
  `UProceduralFoliageComponent`), through the shared actor-name resolution
  `spatial.ground_instances` and `pcg.generate` already use.
- `clearFirst:true` dispatches `ClearProceduralFoliageComponents` before
  `ResimulateProceduralFoliageComponents`, which is the honest way to express "replace this pass"
  and avoids a separate boolean-mode verb.
- The response must keep `create_procedural`'s existing honesty split, which is already correct and
  should be reused verbatim rather than reinvented: `resimulated` reports only that the reflection
  path was reachable and dispatched (the library function is `void`, so there is no sim-success
  return — stated at `FoliageHandler.cpp:1561-1565`), while the placed count comes from the
  before/after delta of `FFoliageInfo::GetPlacedInstanceCount()` across every
  `AInstancedFoliageActor` (`:1499-1521`). Extract that lambda so the two verbs cannot drift about
  what "placed" means.

**Extraction is the bulk of the work and it is small.** Lifting `:1524-1570` plus the counting
lambda at `:1499-1521` into a shared helper leaves `create_procedural` calling the same code it
calls today, and gives the new verb a body that is mostly actor resolution.

**Severity: Medium, argued.** Impact class is the rubric's *"High or Medium: hard blocker with no
workaround (a stub, a missing verb, or rejecting valid input)"* — a missing verb, with the standard
reflection escape hatch positively closed by the CDO refusal at
`ObjectCallFunctionHandler.cpp:107-110` and by the component declaring no `UFUNCTION` at all. It
lands on **Medium** and not High for one reason, stated plainly so a reviewer can disagree with it:
`python.execute` reaches `unreal.ProceduralFoliageEditorLibrary` directly, so a caller is not
absolutely stuck — the rubric's Medium is *"Doable, but only via a documented workaround, a source
dive, or many extra calls"*, and this is the source-dive case. It is explicitly **not** High: nothing
is silently wrong. `foliage.create_procedural` never claims to have re-simulated anything it did not
(`resimulated`'s contract at `:1561-1565` is careful about exactly this), and no response here is a
lie. **Reach modifier declined in both directions, and named:** procedural-foliage authoring is not
an every-session path, so no bump up; but it is not a rare edge path either — it is a whole
documented workflow, and re-simulation is that workflow's inner loop, hit several times per volume
rather than once. Medium stands unmodified.

## Related

- `F-procedural-foliage-simulation-knobs-unreachable` (OPEN, High) — mutually gating: the knobs
  ticket's `property.set`-on-the-generated-assets workaround needs this verb to take effect, and this
  verb's value is mostly the iteration loop those knobs create. A fixer taking both together lands
  a working tuning cycle; either alone lands half of one.
- `B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) — why `foliage.remove` is not the
  clear path, and a warning about what any new clear path must not do.
- `B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — same verb's spawn path; the
  `AProceduralFoliageVolume` at `:1489-1491` lands with bounds extent `(0,0,0)`, so a re-simulate
  verb tested against a volume created today will measure zero placed instances for an unrelated
  reason. Fix or work around that before verifying anything here.
- `B-foliage-create-procedural-empty-callback-noop` — the ticket that put the reflection path into
  the tree in the first place; this ticket asks only that it be reachable twice.

## History
- `#1-resim-path-exists-but-only-at-creation` `OPEN` reporter — Filed after checking whether the
  premise this started from was stale, and it was: the claim that re-simulation *"needs
  `python.execute` into `ProceduralFoliageEditorLibrary`"* no longer describes the tree, because the
  `/Script/FoliageEdit.ProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents`
  reflection path is already present at `FoliageHandler.cpp:1549-1553` and dispatched at `:1566-1569`.
  A narrow gap survives that restatement and is what this ticket is scoped to: the block is inside
  `foliage.create_procedural`'s body, gated on the volume that call just spawned (`:1526`, against
  `:1489-1491`), and grepping the whole `Plugins/PinWright/Source` tree finds `Resimulate` /
  `ProceduralFoliageEditorLibrary` / `AProceduralFoliageVolume` / `UProceduralFoliageComponent` in
  exactly one non-test file, all inside that one handler — so no verb touches an already-existing
  volume. Read the engine header
  (`C:/UE_5.8/Engine/Source/Editor/FoliageEdit/Public/ProceduralFoliageEditorLibrary.h:29-38`): four
  `BlueprintCallable` statics, of which three (`ResimulateProceduralFoliageVolumes`,
  `ClearProceduralFoliageComponents`, `ClearProceduralFoliageVolumes`) have no path at all.
  Confirmed `object.call_function` cannot substitute — the functions are statics, which means
  invoking on the CDO, which `ObjectCallFunctionHandler.cpp:107-110` refuses outright, and
  `Runtime/Foliage/Public/ProceduralFoliageComponent.h` declares no `UFUNCTION` for it to reach on a
  live object instead. `python.execute` remains, which is what holds this at Medium.
- `#2-mesh-writes-are-not-inert` `OPEN` reporter — Encounter from the zone C/D re-speciation
  (`Docs/map/vegetation-style-split.md` § Findings 5), which bounds one claim in `#1`'s body.
  Measured: changing `UFoliageType::Mesh` on an already-simulated type re-points every
  already-spawned instance on the next frame — no resimulate, no reload, and a component census
  reports the new mesh at the old simulation's transforms. So *"nothing then re-runs the simulation
  to consume them, so the write is inert"* is right for the three properties it names
  (`ProceduralScale` / `InitialSeedDensity` / `OverlapPriority`, all simulation inputs consumed
  once) and wrong as a general rule about `_FT_<n>` writes. Mechanism read at HEAD:
  `UFoliageType::PostEditChangeProperty` (`bSourceChanged` at `InstancedFoliage.cpp:1035`, gate at
  `:1042`, loop at `:1044`) notifies every loaded `AInstancedFoliageActor` at `:1048`, reaching
  `FFoliageInfo::NotifyFoliageTypeChanged`
  (`:2185-2188`) and `FFoliageStaticMesh::NotifyFoliageTypeChanged` (`:1449`) →
  `CheckComponentClass` (`:1452`), `UpdateComponentSettings` (`:1453`),
  `BuildTreeIfOutdated` (`:1466`); `PreEditChange` → `NotifyFoliageTypeWillChange` at `:1060-1062`.
  The re-point is notification-driven, so it follows any write that dispatches
  `PostEditChangeProperty` and not one that bypasses it. **Effect on scope: narrows, does not
  close.** A species swap is the one tuning edit that does not need `foliage.resimulate`, which
  makes it a cheap preview; distribution, density, clustering and age still need the simulation to
  re-run, so severity stays **Medium** and the verb is still wanted. **New hazard recorded, marked
  conditional:** `CheckComponentClass` reaches `FFoliageStaticMesh::Reapply` at
  `InstancedFoliage.cpp:1571-1584` when the new mesh implies a different component class — and
  `Reapply` is the reconciler `B-foliage-remove-empties-ledger-not-component` (Critical) names at
  `:1859-1867` as what turns its ledger/component divergence into deletion. A mesh swap is a
  second route into that reconciler; a plain mesh-for-mesh swap need not change the component
  class, so this is flagged to check rather than demonstrated. `encounters` 1 → 2.
