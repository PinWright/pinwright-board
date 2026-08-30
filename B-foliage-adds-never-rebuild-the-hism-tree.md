---
id: B-foliage-adds-never-rebuild-the-hism-tree
title: "foliage.add_instances writes instances the level never draws: the engine add path suppresses the HISM cluster-tree rebuild for the duration of the add and nothing restores it, and every honesty field the namespace has — renderedInstanceCount, ledgerMatchesRendered, expectedDrawnInstances — is computed from an instance ARRAY rather than the built tree, so all three certify a level that shows nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, add_instances, paint, hism, cluster-tree, silent-false-success, ledger-vs-component, third-representation, honesty-fields, readback-blind-spot, behavioural-test-passes-on-defect, capture-verification]
encounters: 2
lastSeen: 2026-08-30T13:10:00Z
---

# Three representations were reported. The frame is a fourth, and it is the one nobody reads.

`B-foliage-remove-empties-ledger-not-component` established that the foliage namespace had two
representations of one instance and only reported one. Its fix added the second
(`renderedInstanceCount`) and a verdict on whether they agree (`ledgerMatchesRendered`); waves 2/3
added a third (`expectedDrawnInstances`, explicitly documented as "the only one about the FRAME").
**There is a fourth, it is the one the viewport draws, and none of the three is it.** All three are
derived from `UInstancedStaticMeshComponent::PerInstanceSMData` — a flat array on the component —
while a HISM draws from its **cluster tree**, and `foliage.add_instances` leaves the tree unbuilt.

Fourteen instances were added, all three fields said fourteen, `ledgerMatchesRendered` said `true`,
and four captures from four camera setups contained no tree at all.

## Measured live

Build: **21:37**, which contains wave 1 (`2ba21649`) and **not** waves 2 and 3. 14 instances of a
1432 uu tree at scale 1.5, `foliage.add_instances`, open terrain.

| what was read | value |
|---|---|
| editor-side ledger (`count`) | 14 |
| `renderedInstanceCount` | 14 |
| `ledgerMatchesRendered` | **true** |
| live `UHierarchicalInstancedStaticMeshComponent::GetInstanceCount()` | 14 |
| component `visible` | true |
| start/end cull distance | 0 / 0 (no cull) |
| materials | bound |
| terrain Z under the row | 347 / 325 |
| instance Z | 333 / 206 (not buried) |
| **four captures** — perspective at 3600 uu, point-blank along the row, top-down perspective, top-down ortho | **nothing at all** |

Then one `c.add_instance()` straight through the component API, at the identical camera pose that
had just photographed an empty field: **all 15 appeared**, mean luminance 0.6359 -> 0.5610. Not the
one new instance — all fifteen, in one step. That is the signature of a cluster-tree build, not of
an instance being added, and it is what identifies the mechanism below rather than merely being
consistent with it.

`expectedDrawnInstances` was **not** in the measured build; the claim about it in
§ *Waves 2/3 added a fourth field with the same blind spot* is source-only.

## Mechanism, re-derived at HEAD (`1a9e5778`)

**The plugin never rebuilds the tree.** `grep -n "Refresh\|MarkRenderStateDirty\|BuildTree"
Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp` returns **zero hits** at HEAD.
Waves 2/3 rewrote ~1014 lines of this file and did not change that.

The add loop is `FoliageHandler.cpp:1876-1893` (registration `:1638`): construct `FFoliageInstance`,
`Info->AddInstance(FoliageType, Instance, nullptr)` at `:1884` (or `:1888` on the create-info
branch), `++Added`, and after the loop `IFA->Modify()` at `:1893`. Nothing else.

The engine side is where the tree is switched off:

| step | site |
|---|---|
| `FFoliageInfo::AddInstance(Settings, Instance, BaseComponent)` -> `AddInstance(Settings, Instance)` | `C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:2328-2337` |
| -> `AddInstances(Settings, { &Instance })` — a **one-element batch** | `:2339-2342` |
| -> `AddInstancesImpl`, which brackets the whole add in `Implementation->BeginUpdate()` / `EndUpdate()` | `:2306-2326` (`BeginUpdate` `:2314`, `EndUpdate` `:2325`) |
| `FFoliageStaticMesh::BeginUpdate` sets `Component->bAutoRebuildTreeOnInstanceChanges = false` | `:1355-1371`, the write at `:1362` |
| `FFoliageStaticMesh::AddInstance` -> `Component->AddInstance(worldTransform, true)` | `:1216-1221` |
| `UHierarchicalInstancedStaticMeshComponent::AddInstance` sets `bIsOutOfDate = true`, appends to `InstanceReorderTable` / `UnbuiltInstanceBoundsList`, then rebuilds **only** `if (bAutoRebuildTreeOnInstanceChanges)` | `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/HierarchicalInstancedStaticMesh.cpp:2468-2499`; `bIsOutOfDate` `:2482`, the gate `:2493-2496` |
| `FFoliageStaticMesh::EndUpdate` **restores the flag and does not rebuild** | `InstancedFoliage.cpp:1373-1388` |

So the flag is `false` for exactly the window in which every instance is added, restored to its
default `true` (`HierarchicalInstancedStaticMesh.cpp:2073`) the moment the window closes, and the
one rebuild the whole sequence would have triggered is the one it skipped. The component exits with
`bIsOutOfDate: true`, `PerInstanceSMData.Num() == 14`, and `NumBuiltInstances` at whatever it was
before — `0` for a freshly created info, which is the measured case.

**The missing call is already in the engine's public surface and the plugin links it.**
`FFoliageInfo::Refresh(bool Async, bool Force)` (`InstancedFoliage.h:377`, body `:2822-2826`) ->
`FFoliageStaticMesh::Refresh` (`:1390-1396`) -> `Component->BuildTreeIfOutdated(Async, Force)`. That
is exactly one call, and it is exactly what `FFoliageInfo::RemoveInstancesImpl` does for itself when
asked: `if (RebuildFoliageTree) { Refresh(true, true); }` at `:2505-2508`.

**`AddInstancesImpl` is the only bracketed mutator in the engine's foliage file that does not end
in a `Refresh`, which is what makes this an API contract the caller owns rather than an engine bug.**
Its siblings all close the same way: `RemoveInstancesImpl` `EndUpdate` `:2503` + `Refresh(true, true)`
`:2507`; `FFoliageInfo::DuplicateInstances` (`:2575-2588`) — which adds through **this same
`AddInstance`** in a loop — `EndUpdate` `:2586` + `Refresh(true, true)` `:2587`. `AddInstancesImpl`
ends at `EndUpdate` `:2325` and stops. So the engine hands the rebuild to whoever called
`AddInstance`, and its own in-file caller does it one line later.

**Which is why `foliage.remove` is not affected and this is a one-verb defect rather than a
namespace one.** Wave 1's `RemoveAllFoliageInstances` helper passes
`Info.RemoveInstances(AllIndices, /*RebuildFoliageTree*/ true)` at `FoliageHandler.cpp:1126`. The
engine's own procedural placement path passes the same: `FEdModeFoliage::AddInstances(World,
DesiredFoliageInstances, OverrideGeometryFilter, /*InRebuildFoliageTree*/ true)`
(`C:/UE_5.8/Engine/Source/Editor/FoliageEdit/Private/ProceduralFoliageEditorLibrary.cpp:72`). Every
engine caller that places foliage rebuilds after it. `foliage.add_instances` is the one that does
not.

**Correction to the brief this was filed from, and it makes the finding wider, not narrower.** The
brief said `foliage.paint` is unaffected because it "routes through `RemoveInstances`". Both halves
are wrong. Paint's two `RemoveInstances` calls (`FoliageHandler.cpp:936`, `:953`) pass
`/*RebuildFoliageTree*/ **false**`, so they rebuild nothing; and they are on the *withdrawal* path,
which a successful paint never reaches. Paint adds through the same `Info->AddInstance` at `:923`.
What actually saves paint is a side effect on a **different** branch: with `surface` supplied it
calls `Info->PostMoveInstances` at `:982`, which goes `FFoliageInfo::PostUpdateInstances`
(`InstancedFoliage.cpp:2528-2561`) -> `Implementation->SetInstanceWorldTransform` (`:2540`) ->
`FFoliageStaticMesh::SetInstanceWorldTransform` (`:1238-1243`) ->
`UHierarchicalInstancedStaticMeshComponent::UpdateInstanceTransform`, whose non-in-place branch
calls `BuildTreeIfOutdated(/*Async*/true, /*ForceUpdate*/false)`
(`HierarchicalInstancedStaticMesh.cpp:2362`) — and in the editor the in-place branch is unreachable,
because `bAllowInPlaceUpdateForRotationOrScaleChange = bIsGameWorld` (`:2331-2335`). Plus
`FFoliageStaticMesh::PostUpdateInstances` -> `Component->MarkRenderStateDirty()`
(`InstancedFoliage.cpp:1250-1255`).

**Consequence: `foliage.paint` WITHOUT `surface` has this defect too.** That branch is the plain
`else` at `FoliageHandler.cpp:986-989` — `AppendPaintedInstanceRow` and nothing else, no
`PostMoveInstances`, no rebuild. It is also the branch every existing behavioural test uses. This
half is **source-only**: unprojected paint was not captured.

## The honesty fields are all computed from the same array

`renderedInstanceCount` comes from `CountRenderedFoliageInstances` (`FoliageHandler.cpp:1094-1096`),
which is `Info.Implementation->GetInstanceCount()` at `:1095`, accumulated at `:1409` / `:1435` and
emitted at `:1464`. Follow it down:

- `FFoliageStaticMesh::GetInstanceCount` -> `Component->GetInstanceCount()`
  (`InstancedFoliage.cpp:1188-1196`)
- `UInstancedStaticMeshComponent::GetInstanceCount` -> **`return PerInstanceSMData.Num();`**
  (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/InstancedStaticMesh.cpp:4844-4847`)

`PerInstanceSMData` is a second bookkeeping array that happens to live on the component. The
component's own count of what is in the tree is a different, adjacent field: `NumBuiltInstances`,
`// The number of instances in the ClusterTree`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/Components/HierarchicalInstancedStaticMeshComponent.h:156-158`).

What the field is documented as, at HEAD, in three places:

- registration summary, `FoliageHandler.cpp:1244`: *"renderedInstanceCount is what the level actually
  draws over the same scope, read off the foliage components"*
- in-code comment, `:1337-1341`: *"What the level DRAWS across the same scope"*
- `Plugins/PinWright/docs/wiki-src/foliage.md:21`: *"`renderedInstanceCount` is what the level
  actually **draws** over the same scope"*

It read 14 while the level drew 0. `ledgerMatchesRendered` (`:1465-1466`) is
`renderedInstanceCount == count + orphanedInstanceCount`, so it compared `Instances.Num()` against
`PerInstanceSMData.Num()` — two arrays a single `AddInstance` writes in lockstep — and returned
`true`. **The verdict field cannot be false for this defect.** It is not that it missed the
divergence; there is no divergence between the two things it compares.

### Waves 2/3 added a fourth field with the same blind spot (source-only)

`expectedDrawnInstances` is documented at `FoliageHandler.cpp:1244` as *"the third representation and
the only one about the FRAME"*. Its value is
`Scope.ExpectedDrawn = FMath::RoundToInt(LedgerCount * Scope.EffectiveScale)`
(`Source/PinWright/Private/Handlers/Environment/ScalabilityDensityCVars.h:155`, emitted `:185`),
where `LedgerCount` is `renderedInstanceCount` (`FoliageHandler.cpp:1469-1475`). It models exactly
one reason the frame can hold fewer instances than the component — the `foliage.DensityScale` cull —
and `UFoliageType::bEnableDensityScaling` defaults `false`
(`InstancedFoliage.cpp:669`) and is never set by the auto-create path, so `EffectiveScale` is 1 and
`expectedDrawnInstances == renderedInstanceCount == 14`. The guard comment at
`ScalabilityDensityCVars.h:188-189` states the assumption outright: *"Nothing to warn about when
nothing in scope is subject to the cull: the ledger count IS what renders."* On this defect that
sentence is false, and the field that claims to be about the frame is off by 100%.

### And the wave-1 regression test reads the same array a third time

`Source/PinWright/Private/Tests/Environment/TestFoliagePlacementBehaviour.cpp` — wave 1 added
`GatherRenderedInstances` (`:220-236`) explicitly so the tests would stop trusting the ledger. Its
read is `Rendered += Component->GetInstanceCount();` at `:234`: `PerInstanceSMData.Num()` again. All
nine count assertions route through `RequireInstanceCount` (`:252`), so **no assertion in the
foliage suite can fail on this defect**, including
`PinWright.foliage.remove.RemovalReachesTheRenderedInstances`, the regression wave 1 wrote for
precisely this class of bug.

Mutator, readback, verdict field, frame field, and regression test: five surfaces, one array.

## Why this is High and not Critical

**The state does not persist and nothing is lost.** `UHierarchicalInstancedStaticMeshComponent::Serialize`
rebuilds before writing: `if (Ar.IsSaving() && Ar.IsPersistent()) { BuildTreeIfOutdated(/*Async*/false,
/*ForceUpdate*/false); }` (`HierarchicalInstancedStaticMesh.cpp:2139-2143`). A save repairs the
component on its way to disk, and the saved level is correct. `PerInstanceSMData` and
`FFoliageInfo::Instances` never diverge here, so there is no reconciler to resolve destructively —
which is the specific clause `B-foliage-remove-empties-ledger-not-component` is Critical on, and it
does not apply. Rated **High** on the rubric's *silent false-success* band instead.

**Both reach modifiers declined.** No upward bump: `foliage.add_instances` is not an every-session
verb. No downward bump either, and this is the reading being rejected — it is not a rare edge path,
it is the verb's **only** path. There is no input to `add_instances` that rebuilds the tree, and
`foliage.paint`'s default (no `surface`) branch is in the same state.

**What makes it High rather than Medium is the verification surface, not the placement.** A caller
who places 14 trees, photographs an empty field, and reads three green fields plus a `true` verdict
has no route to the truth through the RPC surface. `NumBuiltInstances` is not exposed by any verb;
`python.execute` reaching into the component is the only observation path, and it is the one that
found this. The cheap in-session workaround — add one instance through the component API, or save
the level — only works for someone who has already read this ticket, which the rubric does not
credit as a workaround.

## Fix

One line at the end of the add loop, on the API the engine already uses for the same purpose:

```cpp
  IFA->Modify();
  if (FFoliageInfo *Info = IFA->FindInfo(FoliageType)) {
    Info->Refresh(/*Async*/ true, /*Force*/ true);   // InstancedFoliage.h:377
  }
```

matching `RemoveInstancesImpl`'s own `Refresh(true, true)` (`InstancedFoliage.cpp:2507`). Same
addition on `foliage.paint`'s unprojected branch, or unconditionally after its loop — the projecting
branch's rebuild is currently an accident of `PostMoveInstances` and should not stay load-bearing.

**Do not read `FFoliageInfo::AddInstances` (`InstancedFoliage.h:335`) as the fix.** It is worth
taking — it collapses N `BeginUpdate`/`EndUpdate` pairs and N `PreAllocateInstancesMemory` calls into
one (`InstancedFoliage.cpp:2306-2326`), which is a real win on a 14-instance batch and a large one on
a 10,000-instance batch — but it goes through the **same** `AddInstancesImpl` and rebuilds nothing.
Batching without the `Refresh` leaves this ticket entirely unfixed while looking like it addressed it.

**Then make the honesty fields capable of failing.** `renderedInstanceCount` should read the tree,
not the array — `NumBuiltInstances`
(`HierarchicalInstancedStaticMeshComponent.h:158`) alongside `PerInstanceSMData.Num()`, or a
`builtInstanceCount` / `treeIsOutdated` pair beside the existing two, so a caller can distinguish
"stored" from "drawable". Until one of the reported numbers can disagree with the frame, a future
regression in this class lands silently again. `GatherRenderedInstances`
(`TestFoliagePlacementBehaviour.cpp:220-236`) needs the same change, or the suite keeps passing.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* carries the enumeration and it is not
restated here. This ticket is the class's sharpest instance so far, and it extends it by one turn:
in the enumerated cases *the deciding number was never reported*, whereas here **the deciding number
was reported three times, under three different names, and none of the three is it.** Each field was
added by a fix that correctly identified the previous field as too shallow, and each landed one
layer above the thing that renders. That progression is the finding, and it is the reason the fix
above ends with a field change and not just a `Refresh`.

## Cross-links

- **`B-foliage-remove-empties-ledger-not-component`** (IN-REVIEW, Critical) — **not challenged, and
  do not reopen it on this.** Its subject (`foliage.remove` discarding the ledger without touching
  the component) is genuinely fixed: `RemoveAllFoliageInstances` routes through
  `FFoliageInfo::RemoveInstances(..., RebuildFoliageTree=true)` at `FoliageHandler.cpp:1126`, and
  that fix is independently verified. What this ticket says is narrower and belongs to whoever
  closes it: the **verification field** the fix shipped, `renderedInstanceCount`, does not measure
  what its three doc sites say it measures, so a tester using it to close that ticket is checking
  two arrays against each other. Its own § *Fix* asked for exactly the field that was built —
  *"`foliage.get_instances` should report the component's `GetInstanceCount()`"* — so the shortfall
  is in the ask, not in the execution of it.
- **`B-foliage-writes-vetoed-by-scalability-cvars`** — owns `expectedDrawnInstances`,
  `densityScalingEnabled` and `bEnableDensityScaling` being unreachable. The frame-count claim
  quoted above is that ticket's field; recorded here rather than folded in because the cause is a
  different one and its remedy (a cvar readback) does not touch it.
- **`F-foliage-namespace-has-no-behavioural-tests`** (IN-REVIEW) — second live counter-example. The
  first, recorded on `B-foliage-remove-empties-ledger-not-component`, was that its assertions read
  the ledger. Wave 1 re-pointed them at the component and they still cannot fail here.
- **`B-component-mesh-swap-silently-unseats-instances`** (OPEN) — reasons about
  `BuildTreeIfOutdated` on the ISM/HISM write path from the other side (a rebuild that *does* happen
  and is not reported). Whoever touches the tree-state reporting should read both.
- **`F-ism-create-and-clear-scatter`**, **`F-ism-per-instance-transforms`** — any new verb that adds
  instances to an ISM/HISM outside `bAutoRebuildTreeOnInstanceChanges` inherits this exact trap.

## History
- `#1-adds-leave-the-cluster-tree-unbuilt` `OPEN` reporter — Measured on the **21:37** build (wave 1
  `2ba21649`, without waves 2/3); mechanism re-derived against HEAD `1a9e5778` before filing. 14
  instances via `foliage.add_instances`: ledger 14, `renderedInstanceCount` 14,
  `ledgerMatchesRendered` **true**, live `GetInstanceCount()` 14, component visible, cull 0/0,
  materials bound, not buried (terrain Z 347/325 vs instance Z 333/206) — and four captures
  (perspective 3600 uu, point-blank, top-down perspective, top-down ortho) contain nothing. One
  `c.add_instance()` through the component API at the identical pose made **all 15** appear
  (meanLuminance 0.6359 -> 0.5610), which is a tree build rather than an instance add and is what
  identifies the mechanism. **Source at HEAD:** `Refresh` / `MarkRenderStateDirty` / `BuildTree`
  return zero hits in `FoliageHandler.cpp`; the add loop is `:1876-1893` with `Info->AddInstance` at
  `:1884`/`:1888` and `IFA->Modify()` at `:1893`. `FFoliageInfo::AddInstance` funnels into
  `AddInstancesImpl` (`InstancedFoliage.cpp:2306-2326`), whose `BeginUpdate` sets
  `bAutoRebuildTreeOnInstanceChanges = false` (`:1355-1371`) and whose `EndUpdate` restores it
  without rebuilding (`:1373-1388`), so `HISM::AddInstance`'s rebuild gate
  (`HierarchicalInstancedStaticMesh.cpp:2493-2496`) is false for the whole batch and the component
  exits `bIsOutOfDate`. The missing call is `FFoliageInfo::Refresh` (`InstancedFoliage.h:377`), which
  `RemoveInstancesImpl` already makes for itself (`:2505-2508`) — which is why wave 1's
  `foliage.remove` (`FoliageHandler.cpp:1126`, `RebuildFoliageTree=true`) and the engine's own
  procedural path (`ProceduralFoliageEditorLibrary.cpp:72`, same flag) are unaffected. **Brief
  corrected, widening the finding:** `foliage.paint` is NOT immune via `RemoveInstances` — its two
  calls (`:936`, `:953`) pass `RebuildFoliageTree=false` and sit on the withdrawal path. It draws
  only because its `surface` branch calls `PostMoveInstances` (`:982`), which reaches
  `HISM::UpdateInstanceTransform`'s `BuildTreeIfOutdated` (`:2362`) by way of
  `SetInstanceWorldTransform`; the editor never takes the in-place branch
  (`bAllowInPlaceUpdateForRotationOrScaleChange = bIsGameWorld`, `:2331-2335`). So **unprojected
  paint (`:985-988`) has the same defect** — source-only, not captured. **Honesty-field half:**
  `renderedInstanceCount` is `Info.Implementation->GetInstanceCount()` (`FoliageHandler.cpp:1095`)
  -> `Component->GetInstanceCount()` (`InstancedFoliage.cpp:1188-1196`) ->
  `PerInstanceSMData.Num()` (`InstancedStaticMesh.cpp:4844-4847`), a second array, not the tree; the
  component's tree count is the adjacent `NumBuiltInstances`
  (`HierarchicalInstancedStaticMeshComponent.h:156-158`). Three doc sites say it is what the level
  draws (`FoliageHandler.cpp:1244`, `:1337-1341`, `docs/wiki-src/foliage.md:21`).
  `ledgerMatchesRendered` compares two arrays one call writes together, so it is structurally
  incapable of being false here. Wave 1's own `GatherRenderedInstances`
  (`TestFoliagePlacementBehaviour.cpp:220-236`, read at `:234`) reads the same array, so no
  assertion in the foliage suite fails on this. **Source-only, not in the measured build:**
  `expectedDrawnInstances` (`ScalabilityDensityCVars.h:155`, `:185`) is `renderedInstanceCount` times
  a density scale that is 1 on a type whose `bEnableDensityScaling` defaults false
  (`InstancedFoliage.cpp:669`), so the field documented as "the only one about the FRAME"
  (`FoliageHandler.cpp:1244`) would also have reported 14. **Rated High, not Critical:** the tree is
  derived and `HISM::Serialize` rebuilds it on save (`HierarchicalInstancedStaticMesh.cpp:2139-2143`),
  so nothing is lost and the saved level is correct — the data-loss clause that carries
  `B-foliage-remove-empties-ledger-not-component` does not apply. Both reach modifiers declined; the
  downward one explicitly, because this is the verb's only path and not an edge case. **Explicitly
  NOT proposing a reopen of `B-foliage-remove-empties-ledger-not-component`** — its own fix is real
  and verified; only the verification field it shipped is indicted, and its § *Fix* asked for exactly
  that field. Dedup: searched the board for `BuildTree`, `cluster tree`, `bAutoRebuildTree`, `never
  drawn`, `add_instances`, and every `B-foliage-*` / `F-foliage-*` / `F-ism-*` file. Nearest
  neighbours are `B-component-mesh-swap-silently-unseats-instances` (a rebuild that happens and is
  unreported) and `B-ism-undo-record-unsafe` (forced rebuild on the undo path); neither covers an
  add that never rebuilds. Nothing on the board mentions `NumBuiltInstances`.
- `#2-citation-corrected-unprojected-paint-branch` `OPEN` reporter — Citation fix only, no change of
  claim, no status change. `#1` and the body cited `FoliageHandler.cpp:985-988` for `foliage.paint`'s
  unprojected `else` branch; `:985` is the last line of the PROJECTING branch
  (`AppendPaintedInstanceRow(..., &Seat)`) and the range straddled the boundary. The correct range is
  **`:986-989`** — `} else {` at `:986`, `AppendPaintedInstanceRow(..., /*Seat*/ nullptr)` at
  `:987-988`, `}` at `:989` — and the body now reads that. Everything the entry says about that
  branch (no `PostMoveInstances`, no rebuild, so it carries this defect) is unchanged and re-checked
  against those lines. Recorded rather than edited into `#1` because history is append-only.
- `#3-ism-actor-instance-count-is-the-ledger-itself` `OPEN` reporter — **A third route to
  "`ledgerMatchesRendered` cannot be false", source-only, re-derived at HEAD `1a9e5778`. No status
  change, nothing above `## History` touched, and `#1`'s two routes are not restated.** Both routes
  already on this ticket are about the **StaticMesh** representation. There is a third that needs no
  mutator at all and holds at rest, because on the **ISM-actor** representation the field compares
  the ledger with itself. `FFoliageImpl::GetInstanceCount()` is pure virtual
  (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/InstancedFoliage.h:219`) with three
  implementations, chosen by `FFoliageInfo::GetImplementationType`
  (`Private/InstancedFoliage.cpp:2132-2151`) and instantiated in `CreateImplementation`
  (`:2074-2092`): `UFoliageType_InstancedStaticMesh` -> `EFoliageImplType::StaticMesh` (`:2134-2137`);
  `UFoliageType_Actor` with `bStaticMeshOnly` -> `ISMActor` (`:2141-2144`); otherwise -> `Actor`
  (`:2145-2148`). The three return three different quantities. `FFoliageStaticMesh::GetInstanceCount`
  returns `Component->GetInstanceCount()` = `PerInstanceSMData.Num()`
  (`InstancedFoliage.cpp:1188-1196`), a **second** array — the one this ticket's body indicts.
  `FFoliageActor::GetInstanceCount` returns `ActorInstances.Num()` (`Private/FoliageActor.cpp:118-121`),
  a genuinely independent array of spawned `AActor*` (`FoliageActor.h:12`) that **can** diverge from
  the ledger. **`FFoliageISMActor::GetInstanceCount` is `return Info->Instances.Num();` — that is the
  entire body — at `Private/FoliageISMActor.cpp:237-240`.** `Info` is the base's `FFoliageInfo* Info`
  (`InstancedFoliage.h:210-211`), the owning info handed in at construction (`FoliageISMActor.h:18-19`,
  ctor init `InstancedFoliage.h:197-201`), so `Info->Instances` is literally the ledger that
  `instances[]` and `count` are built from: this representation holds **no second count anywhere**.
  Its siblings are the same pass-through — `GetInstanceWorldTransform` also reads `Info->Instances[...]`
  (`FoliageISMActor.cpp:286`), and `BeginUpdate`/`EndUpdate` (`:269-277`) delegate to
  `AISMPartitionActor` and never touch `bAutoRebuildTreeOnInstanceChanges`, so this ticket's
  tree-suppression mechanism is representation-specific too and was not exercised on this path.
  **Connecting the engine tautology to the plugin field:** `CountRenderedFoliageInstances`
  (`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1094-1096`) is
  `Info.Implementation->GetInstanceCount()` at `:1095`, accumulated at `:1408-1409` (filtered branch)
  and `:1434-1435` (unfiltered), emitted as `renderedInstanceCount` at `:1464`; the verdict at
  `:1465-1466` is `RenderedInstanceCount == InstancesArray.Num() + OrphanedInstanceCount`, where
  `InstancesArray` is filled from `Info->Instances` (`:1411-1415`) and orphans are counted off the
  same array (`:1438`). For an ISM-actor info **every term on both sides is `Info->Instances.Num()`**,
  so that info contributes exactly zero possible divergence, and a scope in which every info is
  ISM-actor answers `ledgerMatchesRendered: true` by identity — at rest, with no write having
  happened. **The dispatch that lands there is deliberate and its reasoning is sound:** the comment at
  `:1092-1093` chose `Implementation->GetInstanceCount()` over `GetComponent()->GetInstanceCount()`
  *"because GetComponent() is null for every non-static-mesh impl type, which would read as 'draws
  nothing'"*, which is correct — the cost is that on one of the three impls the honest-looking call
  returns the ledger. **And the tautology is inherited, not introduced.** `:1091` names the quantity
  as *"the exact quantity FFoliageInfo::CheckValid compares Instances.Num() against"*, and that
  assertion is `check(Instances.Num() == Implementation->GetInstanceCount())`
  (`InstancedFoliage.cpp:2200`) — the same invariant `FoliageHandler.cpp:1461-1462` and
  `docs/wiki-src/foliage.md:21` cite as the thing `ledgerMatchesRendered` reports on. On ISMActor the
  engine's own check is `x == x`, so the plugin field is not weaker than the engine assertion it
  mirrors; it is exactly as empty. **Bound on the claim, recorded so this ticket is not over-claimed:
  the field is NOT useless everywhere, and it has caught a real defect.** A level-wide
  `foliage.get_instances` on `/Game/Maps/PW_VegetationTest` against the **13:32** build (plugin
  `d8f1bc32`, confirmed an ancestor of HEAD `1a9e5778`) returned `count 23351`,
  `renderedInstanceCount 23347`, `orphanedInstanceCount 0`, **`ledgerMatchesRendered: false`** — four
  ledger-only ghosts of one `Auto_S_Bird_Of_Paradise...Var10_lod0` type, all zero-rotator and one at
  an unprojected `z 2800`; `foliage.remove` on that type cleared them and the field went `true`.
  Source of record: `X:/src/unreal/EAContentExamples58/Docs/map/vegetation-agent-brief.md`
  § *New and load-bearing: `foliage.get_instances` publishes `ledgerMatchesRendered`* (project commit
  `7d629ad9`). That level is the **StaticMesh** impl on this ticket's own evidence — its § *Measured
  live* read `UHierarchicalInstancedStaticMeshComponent::GetInstanceCount()` there, and the brief's
  removal check moved a component's `GetInstanceCount()` 12 -> 0 — i.e. the representation where the
  two arrays are separable by a writer that touches one and not the other. **So the tautology
  recorded here is scoped to `EFoliageImplType::ISMActor` and does not generalise to the namespace.**
  **What it changes about the ticket:** the § *Fix* ask — read the tree, `NumBuiltInstances` alongside
  `PerInstanceSMData.Num()` — is written for the StaticMesh impl and does not reach this one at all.
  An ISM-actor's drawn instances live on the partition actor's ISM components behind
  `FISMClientHandle` (`FoliageISMActor.h:31`; adds go `GetIFA()->AddISMInstance(ClientHandle, ...)`,
  `FoliageISMActor.cpp:253-256`), which `foliage.get_instances` never reads. Whoever makes the
  honesty fields capable of failing therefore has to do it **per representation**, or publish the
  impl type beside the counts so a caller can tell which of the three numbers they were handed —
  otherwise the fix lands on StaticMesh and `ledgerMatchesRendered` stays structurally `true` on
  ISM-actor foliage with nothing saying so. **Source-only:** no ISM-actor foliage type was exercised
  live; every citation above is a source derivation at HEAD, and all the engine lines sit outside the
  plugin, so they are identical in the 13:32 build. `encounters` 1 -> 2, `lastSeen` refreshed; status
  left `OPEN`.
- `#4-rebuild-and-a-field-that-can-fail` `IN-REVIEW` developer — "Both halves fixed. **(1) The
  rebuild.** New `Source/PinWright/Private/Handlers/Environment/FoliageClusterTreeState.h` carries
  `RebuildFoliageClusterTreeAfterAdd(FFoliageInfo&)`, which is `Info.Refresh(Async, Force)` behind an
  `IsInitialized()` guard (`Refresh` opens `check(Implementation.IsValid())`). Called after the add
  loop in `foliage.add_instances` (`FoliageHandler.cpp`, after `IFA->Modify()`, re-resolving the info
  through `FindInfo` because the loop's own handle is scoped to the iteration) and
  **unconditionally** after `foliage.paint`'s loop — on the projecting branch too, because that
  branch's rebuild was the `PostMoveInstances` side effect this ticket identified and it must not stay
  load-bearing. **Deviation from the ticket's § *Fix*, stated because it is deliberate: `Async=false`,
  not the `Refresh(true, true)` the engine's in-file callers use.** The same response now publishes
  the measured `builtInstanceCount`, and an async build returns with `bIsOutOfDate` still set — a
  caller would read a warning about a scatter that is merely a few frames early, which reproduces this
  ticket's shape rather than closing it. The engine already forces sync for a component's FIRST build
  (`bForceSync = NumBuiltInstances == 0 && !World->HasBegunPlay()`,
  `HierarchicalInstancedStaticMesh.cpp:2812`), so this only changes the append case. `FFoliageInfo::
  AddInstances` batching was NOT taken — the ticket is explicit that it goes through the same
  `AddInstancesImpl` and would look like a fix while changing nothing. **(2) The field that can fail.**
  `builtInstanceCount` = `Component->NumBuiltInstances`, `clusterTreeUpToDate` =
  `Component->IsTreeFullyBuilt()`, plus `clusterTreeWarning`, published by
  `AddFoliageClusterTreeReport` on `add_instances`, `paint` AND `get_instances`. House convention
  followed throughout: measured value published, the requested counter named separately as
  `RequestedFieldName` (`instances_count` / `instancesPlaced` / `renderedInstanceCount`), **OMITTED
  rather than zeroed** in two distinct cases — no component at all (a type before its first add), and
  any info in scope whose impl is not StaticMesh — and a warning when the layers disagree. The
  non-mesh omission is `#3`'s ISM-actor concern answered directly: `builtInstanceCount` is absent with
  a `clusterTreeWarning` naming why, rather than silently short. `get_instances` **reports and does
  not repair** — a read verb that rebuilt a stale tree would hide the writer that broke it.
  **`ledgerMatchesRendered`, `renderedInstanceCount` and `expectedDrawnInstances` are untouched**, per
  this ticket's own instruction not to reopen `B-foliage-remove-empties-ledger-not-component`; the
  new field sits beside them and the warning text names `ledgerMatchesRendered` explicitly as the
  verdict that stays `true` throughout, so the blind spot is stated on the wire rather than silently
  patched. **`builtInstanceCount` vs `expectedDrawnInstances`, kept apart on purpose.** They are
  adjacent layers and the engine keeps both numbers next door to each other:
  `NumBuiltInstances` is PRE-density and `NumBuiltRenderInstances` is post-density
  (`HierarchicalInstancedStaticMeshComponent.h:156-161`). Reading `NumBuiltInstances` is what keeps
  the density cull reported ONCE, by `expectedDrawnInstances`, instead of twice. A caller tells them
  apart by what a shortfall means: `builtInstanceCount` short = **stale tree**, fixed by rebuilding or
  saving; `expectedDrawnInstances` short = **scalability setting**, fixed by raising
  `foliage.DensityScale`. Neither is a camera figure. Stated in the header, in all three registration
  summaries, and in a new `##` section of `Docs/wiki-src/foliage.md` (placed above the first `###`, so
  it renders on the namespace page). **(3) Regression test**, new file
  `Source/PinWright/Private/Tests/Environment/TestFoliageClusterTreeRebuild.cpp`, two ids:
  `PinWright.foliage.add_instances.AddedInstancesReachTheClusterTree` and
  `PinWright.foliage.paint.UnprojectedPaintReachesTheClusterTree`. **Asserts `NumBuiltInstances` and
  `IsTreeFullyBuilt()` read LIVE off the component — nothing else in the suite would go red on this
  defect.** Deliberately asserts the array and ledger layers alongside, labelled as the checks that
  agreed throughout, so the blind spot is documented in the test rather than merely avoided; a reader
  can see which layer the coverage was missing. Counterfactuals: reverting the `add_instances`
  `Refresh` leaves 0 built against 14 stored; reverting paint's leaves the unprojected branch at 0;
  switching to `Async=true` is caught by the second-batch (append) assertions, which are the case the
  engine does NOT force sync; publishing `builtInstanceCount` from `GetInstanceCount()` instead makes
  the response assertions pass while the live-component ones stay red. New file rather than an
  addition to `TestFoliagePlacementBehaviour.cpp` because four other agents were editing
  `FoliageHandler.cpp` concurrently; helpers carry a `Ctr` prefix and were grep-verified unique across
  `Source/` for the Unity build. `Content/Python/check_test_ids.py` re-run: `SCANNED 4790 id(s), 4790
  unique ... CLEAN`. **NOT COMPILED AND NOT RUN** — the orchestrator builds and runs the suite after
  all agents return; the defect was verified in this checkout statically (`Refresh` / `BuildTree` /
  `MarkRenderStateDirty` / `NumBuiltInstances` still return **zero** hits in `FoliageHandler.cpp` at
  local `d8f1bc32`, and the whole engine chain from `AddInstancesImpl` through `BeginUpdate`'s
  `bAutoRebuildTreeOnInstanceChanges = false` to an `EndUpdate` that restores without rebuilding was
  re-read at `C:/UE_5.8/Engine/Source`) rather than driven live, because the shared editor was in use
  by fifteen concurrent agents and one of them is fixing an editor-killing `foliage.add_type` crash.
  **Access check done rather than assumed:** `NumBuiltInstances` and `bIsOutOfDate` sit in the
  `GENERATED_UCLASS_BODY()` section, whose generated legacy body ends in `public:` (verified in
  `HierarchicalInstancedStaticMeshComponent.generated.h`), so the reads need no accessor. No error
  code added, so `FoliageHandler.cpp` keeps its zero `ErrorCodes::` references and its raw-literal
  style intact. Docs: the new `##` section plus per-verb paragraphs on `get_instances`,
  `add_instances` and `paint` in `Docs/wiki-src/foliage.md`."
