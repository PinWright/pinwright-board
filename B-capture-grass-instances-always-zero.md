---
id: B-capture-grass-instances-always-zero
title: "the capture response's grass.instances counts PerInstanceSMData through GetInstanceCount(), which landscape grass never populates — 149 grass components rendering a dense carpet report instances: 0, so the one field that says HOW MUCH grass was built is structurally incapable of reporting any"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture_open_level, landscape, grass, vegetation, verification-field, structurally-constant-field, hism, per-instance-render-data, silent-false-negative, getinstancecount]
encounters: 2
lastSeen: 2026-08-30T19:00:00+05:00
---

# The grass block counts components correctly and instances never

`viewport.grass` reports `components`, `componentsBefore`, `pendingComponents`, `pendingTasks`
and `instances`. Four of those are measured. The fifth is always zero, for every landscape grass
type, in every capture — and it is the only one that answers *how much grass is standing there*.

`Handlers/Render/LandscapeGrassSettle.cpp:40-50`, inside
`PinWrightCaptureGrassMeasureProxies`:

```cpp
            for (const FCachedLandscapeFoliage::FGrassComp& GrassComp : Proxy->FoliageCache.CachedGrassComps)
            {
                ++OutComponents;
                ...
                if (const UHierarchicalInstancedStaticMeshComponent* Foliage = GrassComp.Foliage.Get())
                {
                    OutInstances += static_cast<int64>(Foliage->GetInstanceCount());
                }
            }
```

`UInstancedStaticMeshComponent::GetInstanceCount()` returns `PerInstanceSMData.Num()`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/InstancedStaticMesh.cpp:4844-4847`; the method is
non-virtual, declared at `Components/InstancedStaticMeshComponent.h:435`, and
`UHierarchicalInstancedStaticMeshComponent` does not override it). **Landscape grass does not put
its instances there.** The grass system builds the render buffer directly and the editor-side
`PerInstanceSMData` array stays empty, so the accessor answers 0 while the component draws
thousands.

## Measured

`PW_VegetationTest` on the 13:32 build (editor pid 18592), three `render.capture_open_level`
calls at three different poses:

| pose | `components` | `componentsBefore` | `buildMs` | `instances` |
|---|---|---|---|---|
| `(0, 0, 0)` | 190 | 136 | 50.9 | **0** |
| `(18000, 18000, 2000)` | 155 | 149 | 18.2 | **0** |
| `(-18000, -18000, 2000)` | 184 | 149 | 32.8 | **0** |

The component counts move per pose and are clearly real. `instances` is 0 in all three.

**Independent of the capture verb entirely**, through `python.execute` against the live actor:
`Landscape_0` carries **149** `UHierarchicalInstancedStaticMeshComponent`s whose static mesh is
`S_Nordic_Coastal_Groundcover_vlqkdeaja_Var6_lod0`, and `get_instance_count()` summed over all
149 is **0**.

**And the frame is full of grass.** The `(-18000, -18000, 2000)` capture
(`Saved/Screenshots/OpenLevel/pw_grass_poseC_posed.png`, exposure pinned at EV100 -0.5) was
looked at, not just measured: a dense grass carpet across the whole meadow, wildflowers, rocks
seated in it, trees on the ridge. `meanLuminance 0.6302`, `litPixelFraction 1.0`. So the reading
is not "there is no grass" — it is an instrument that cannot see grass.

## Why this matters more than a cosmetic zero

`instances` is the field a caller reads to decide whether a capture is worth trusting, and it
reads exactly the same — `0` — in the three cases a caller most needs to tell apart:

1. grass built correctly and densely (measured above);
2. grass genuinely absent, e.g. no grass type assigned;
3. grass components created but empty.

`components` partly covers case 2, but a component count says nothing about density, and the
`grassWarning` texts (`LandscapeGrassSettle.cpp:277`, `:296`, `:322`) are gated on
`builtForPose` / `settled` / reach, not on instance count — so nothing else in the block fills
the gap. A caller doing what the block exists for gets a number that is always the alarming one.

## Same root cause as an already-filed ticket, in its most extreme form

`B-foliage-adds-never-rebuild-the-hism-tree` (High) established that `GetInstanceCount()` reads
`PerInstanceSMData.Num()` — an array, not the built cluster tree — and that this makes
`foliage.get_instances`'s `ledgerMatchesRendered` unable to be false on the add path. **This is
the same accessor failing harder**: there the two arrays are written together so the number is
merely uninformative; here the array is never written at all, so the number is a constant 0
against arbitrarily much rendered geometry. Worth reading together; the fix for one does not fix
the other, because the call sites and the correct substitute differ.

A third instance of the same accessor's limits, found in the same session and recorded on
`B-foliage-remove-empties-ledger-not-component` `#4`: `FFoliageISMActor::GetInstanceCount()`
returns `Info->Instances.Num()` (`C:/UE_5.8/.../FoliageISMActor.cpp:237-240`), which makes the
same comparison tautological for ISM-actor foliage. Three different subsystems, one accessor,
three different wrong answers.

## Fix

`GetInstanceCount()` is the wrong instrument for grass. The count that exists is on the render
data: `UInstancedStaticMeshComponent::PerInstanceRenderData->InstanceBuffer.GetNumInstances()`
when `PerInstanceRenderData` is valid, which is what the grass path populates. Suggested shape,
preserving this project's rule that an unmeasurable number is omitted rather than guessed:

- read the render-data count when present, fall back to `GetInstanceCount()` when it is not;
- if neither is readable, **omit `instances`** and add a `warnings[]` line, rather than emitting
  `0` — a `0` that means "not measured" is the defect this ticket is about, and re-emitting it
  from a different branch would not be an improvement;
- report which source was used, the way the capture block already reports `adaptedSource` for
  exposure, so a reader can tell a measured 0 from an unmeasurable one.

**Not verified:** that `PerInstanceRenderData` is non-null and correctly populated on these grass
components at the moment `PinWrightCaptureGrassMeasureProxies` runs. That is the one thing a
fixer must check first, because the measurement above establishes only that
`PerInstanceSMData` is empty — it does not establish where the real count lives. Read it before
writing the fallback.

## Cross-links

- `B-capture-open-level-pose-params-photograph-stale-grass` (DONE) — the ticket whose fix built
  this block. Its subject (pose drives the grass build) is verified and unaffected; this is a
  defect in one field of the reporting the fix added, found while verifying it, and it is filed
  separately rather than reopening that ticket.
- `B-foliage-adds-never-rebuild-the-hism-tree` (High) — same accessor, same root cause, different
  call site and different consequence. Read together.
- `B-ortho-capture-renders-no-landscape-grass` (IN-REVIEW) — the ortho path's grass problem is
  structural (an ortho view contributes nothing to `ViewLocationsRenderedLastFrame`) and separate;
  but it emits the same block (`OrthoTileCaptureHandler.cpp:832`, `:952`), so it carries this
  field and this defect too.

## Severity

**High.** Impact class: a verification field that reports a constant value indistinguishable from
failure, on the surface a caller uses to decide whether a render is trustworthy. Not Critical —
nothing is corrupted, no data is lost, and the surrounding fields (`components`, `builtForPose`,
`buildMs`) do let a careful caller reach the right conclusion, so the false signal is
recoverable. Not Medium — there is no workaround through the RPC surface, since no verb exposes
the grass components' render-data counts, and the field's failure mode points the wrong way
(it under-reports, so it reads as "the capture is bad" on a good capture).

## History
- `#1-grass-instances-structurally-zero` `OPEN` reporter — Found while verifying
  `B-capture-open-level-pose-params-photograph-stale-grass` on the 13:32 build (editor pid
  18592). Three captures at three poses reported `components` 190/155/184 against
  `componentsBefore` 136/149/149 with `buildMs` 50.9/18.2/32.8 — all moving, all plausible — and
  `instances: 0` in every one. Confirmed independently of the capture verb through
  `python.execute`: `Landscape_0` holds 149 grass HISMs on
  `S_Nordic_Coastal_Groundcover_vlqkdeaja_Var6_lod0` and `get_instance_count()` sums to 0 across
  all of them. Confirmed visually rather than numerically that grass is in fact present:
  `pw_grass_poseC_posed.png` (exposure pinned EV100 -0.5) shows a dense carpet over the whole
  meadow. Mechanism re-derived at HEAD `1a9e5778`: `LandscapeGrassSettle.cpp:40-50` sums
  `Foliage->GetInstanceCount()`, which is `PerInstanceSMData.Num()`
  (`InstancedStaticMesh.cpp:4844-4847`, non-virtual, not overridden by HISM), an array the
  landscape grass path does not populate. Filed as its own ticket rather than reopening the
  ticket whose fix introduced the block, because that fix's subject is verified and this is a
  defect in one reported field. Cross-linked to `B-foliage-adds-never-rebuild-the-hism-tree` as
  the same accessor's already-filed failure in a milder form. **Not verified and left to the
  fixer:** where the real count lives at measure time — `PerInstanceRenderData` is the proposed
  source but was NOT read. No fix attempted; nothing under `Plugins/PinWright/Source/` was edited.
- `#2-the-docs-instruct-callers-to-trust-this-field` `OPEN` reporter — **Status and severity deliberately unchanged; no code touched.** The documentation half of this ticket, found while filing `B-capture-docs-prescribe-retired-grass-wait` and appended here rather than left there, because the person fixing the field is the person who has to fix the sentence. `Docs/wiki-src/render.md:56` at plugin HEAD `1a9e5778` (generated `Saved/PinWright/wiki/render.check-the-view-mode-before-you-trust-a-capture.md:40`), verbatim: *"**Read `settled` before concluding \"no vegetation here\".** `settled: true` with `instances: 0` is a measurement — that ground really is bare. `settled: false` means the build had not finished when the shutter fired and comes with a `grassWarning`; those pixels may be missing grass that exists. The two are the same picture and opposite conclusions, which is why the block is unconditional."* So the `render` namespace page — the page a caller reads **before** capturing — nominates this field as the evidence for a positive conclusion about the level, and does it in the one sentence that tells the reader how to tell bare ground from an unfinished build. **The structural zero re-derived at HEAD rather than relayed from `#1`:** `PinWrightCaptureGrassMeasureProxies` walks `Proxy->FoliageCache.CachedGrassComps` at `Source/PinWright/Private/Handlers/Render/LandscapeGrassSettle.cpp:41-52`, and the only thing it adds to `OutInstances` is `Foliage->GetInstanceCount()` at `:50`, inside the `UHierarchicalInstancedStaticMeshComponent` guard at `:48`; `UInstancedStaticMeshComponent::GetInstanceCount()` is `return PerInstanceSMData.Num();` — `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/InstancedStaticMesh.cpp:4844-4847`, declared `Classes/Components/InstancedStaticMeshComponent.h:435`. Still resolves, still a single-expression accessor over the array landscape grass never populates. (`#1` cites `:40-50`, which brackets the loop; at HEAD the call itself is `:50`.) **Why this makes the defect worse rather than merely documenting it.** `#1` establishes that the field is structurally incapable of reporting grass — a false negative. The sentence above converts that false negative into an affirmative statement about the world: a caller who follows it reads `settled: true, instances: 0` off a frame with a dense carpet in it and concludes the ground is bare. Nothing else in the block contradicts them, because `components` is the field that would have, and the sentence does not mention it. The sentence is also precise and correct about the `settled: false` half, which is exactly what earns the `settled: true` half the reader's trust. **Ask, tied to this ticket's own Fix section:** whichever of the three shapes is taken, `Docs/wiki-src/render.md:56` changes with it in the same commit — if `instances` is omitted when unmeasurable, the sentence must stop keying "no vegetation here" on it at all; if a source-disclosure field is added the way the capture block reports `adaptedSource`, the sentence should key on that instead; and until the field is trustworthy the sentence should name `components` as the number to read. **Cross-link, both directions:** `B-capture-docs-prescribe-retired-grass-wait` (OPEN, Medium) owns the *other* documentation defect on this same overlay page — `render.md:127-129` still prescribes `editor.set_camera` plus a 60-150 s wait plus repeat-until-stable captures for a grass amortization the capture's own settle force-syncs past — and records this finding in its § *Noted here, deliberately NOT filed* as belonging here. Two independent doc defects about the same `grass` block on one page, ~70 lines apart; whoever sweeps `render.md` should read both, and neither absorbs the other because one is a broken field's documentation and the other is retired guidance about a fixed verb. `encounters` 1 -> 2 per the schema's dedup-append rule and `lastSeen` refreshed; **neither is a severity input** — this remains **High** on `#1`'s argument, unchanged, and no status change is proposed: the field is still broken and the ticket is still `OPEN`.
- `#3-count-the-render-side-instances` `IN-REVIEW` developer — "**The ticket's proposed source does not exist on UE 5.8 and was not used.** `PerInstanceRenderData` is a bare forward declaration in `Classes/Components/InstancedStaticMeshComponent.h:26` and no member of `UInstancedStaticMeshComponent`; the only surviving mention of `PerInstanceRenderData->InstanceBuffer.GetNumInstances()` in the engine is inside an `#if 0` block (`HierarchicalInstancedStaticMesh.cpp:2615`, whose comment nevertheless states the mechanism: *'Non-nanite HISMs that were built with AcceptPrebuiltTree won't have PerInstanceSMData'*). The `#1` Fix section's first bullet would not have compiled. **Where the count actually lives, read rather than assumed** — the one thing `#1` left to the fixer. `ALandscapeProxy::ProcessAsyncGrassInstanceTasks` takes `NumBuiltRenderInstances = Inner.Builder->InstanceBuffer.GetNumInstances()` and calls `UGrassInstancedStaticMeshComponent::AcceptPrebuiltTree` with it (`LandscapeGrass.cpp:3370`, `:3387`) — the ONLY path by which a grass component ever receives instances. That function opens with `check(!PerInstanceSMData.Num())` (`Runtime/Foliage/Private/InstancedGrass.cpp:52`), so `#1`'s zero is not merely empirical, it is asserted by the engine; it then sets `NumBuiltRenderInstances` and `InstanceCountToRender` from the argument (`:55`, `:65`) and the component publishes the former through its override of the public virtual `UInstancedStaticMeshComponent::GetNumRenderInstances()` — *'Number of instances in the render-side instance buffer'*, `InstancedStaticMeshComponent.h:639`, overridden at `GrassInstancedStaticMeshComponent.h:22`. Verified present with an identical signature on 5.3, 5.4, 5.5, 5.6, 5.7 and 5.8, so the fix carries no version gate and adds no row to `Docs/engine-version-support.md`. **The fix.** New `PinWrightCaptureGrass::GrassComponentInstanceCount()` (declared in `LandscapeGrassSettle.h`, defined in the `.cpp`) returns `GetNumRenderInstances()` when positive and falls back to `GetInstanceCount()` otherwise. The fallback is load-bearing, not defensive: HISM's own override of that virtual is `SortedInstances.Num()` (`HierarchicalInstancedStaticMeshComponent.h:342`), which is zero on a hand-authored component whose cluster tree has not been built while `PerInstanceSMData` holds real instances — neither number alone is right for both kinds of component. **A SECOND, UNFILED INSTANCE OF THE SAME DEFECT WAS FOUND AND FIXED IN THE SAME BLOCK, and it is worse than a wrong number.** `MeasureGrassFrameReach` — the `grass.reach` sub-block shipped by `B-ortho-capture-renders-no-landscape-grass` — opened its per-component loop with `if (!Foliage || Foliage->GetInstanceCount() <= 0) { continue; }` and summed the same accessor into `InstancesInReach`. Every real grass component therefore failed that guard, `bAnyComponentSeen` stayed false, the function returned at its `!bAnyComponentSeen` early exit, and `bReachMeasured` was left false — so **the entire `reach` sub-block was silently absent from every capture of every real landscape grass**, on exactly the content it was written for. Both call sites now go through the shared helper. That ticket is IN-REVIEW and its own subject (an ortho pose drives its own grass build) is unaffected; this is recorded here rather than reopening it, the same way `#1` was filed rather than reopening the ticket that built the block. **The neighbouring counts were checked for the same class of defect and are clean.** `components` / `componentsBefore` count `FoliageCache.CachedGrassComps` entries directly, `pendingComponents` reads the engine's own `FGrassComp::Pending` flag (cleared at `LandscapeGrass.cpp:3399`), and `pendingTasks` is `AsyncFoliageTasks.Num()` — none goes through an accessor, none can be structurally constant, and `#1`'s own measurements (190/155/184 against 136/149/149) already show them moving. One unrelated observation, NOT fixed and NOT filed as it is outside this ticket: a cache entry whose `FAsyncGrassBuilder` had `bHaveValidData == false` never gets a task (`LandscapeGrass.cpp:3247-3259`), so nothing ever clears its `Pending`, and the trim loop's `bOld` requires `!Pending` — such an entry pins `settled` false indefinitely. **`instances` is now OMITTED, never guessed.** A cache entry whose component weak pointer has already expired cannot be read at all, and the engine treats that state as real (`UpdateGrass`'s trim loop lists `!Used` as a reason to drop an entry, `LandscapeGrass.cpp:3286-3298`, so the entry outlives the component until the next update). Those entries are counted into a new `UnreadableComponents`; `bInstancesMeasured` is `UnreadableComponents == 0`; when it is false the response omits `instances`, publishes `unreadableComponents` and carries a `grassWarning` telling the caller not to read the absence as zero and to read `components` instead. No `instancesSource` disclosure field was added, deliberately: `PerInstanceSMData` is empty on every component the landscape foliage cache can hold, so such a field would itself be a near-constant string — the same class of defect this ticket is about. Presence-or-absence of `instances` carries the distinction instead. **Doc half of the ticket (`#2`) fixed in the same change.** `Docs/wiki-src/render.md:56`'s sentence is kept — `settled: true` with `instances: 0` really is a bare-ground measurement once the count works — and a new paragraph beneath it tells callers to branch on whether `instances` is present rather than on its value, names the old `PerInstanceSMData` read as the reason it was always zero, and points at `components` for the absent case; the `viewport` field table row at `:42` now marks `instances` conditional. **Regression tests**, `Source/PinWright/Private/Tests/Render/TestCaptureGrassInstanceCount.cpp`, both failing on the pre-fix expression: `PinWright.render.grass_instances.LandscapeGrassInstancesAreCounted` builds a component in exactly the state `AcceptPrebuiltTree` leaves one in (class resolved by `FindObject` per the plugin's MinimalAPI convention, `PerInstanceSMData` untouched, `NumBuiltRenderInstances` = 41237) and asserts the old accessor reads 0 on it while the new count reads 41237 and is non-zero — plus both counter-directions, an authored HISM whose instances live only in `PerInstanceSMData` and a grass component that built nothing, so the fix can be neither 'always the render count' nor any non-zero constant. `PinWright.render.grass_instances.UnreadableCountIsOmittedNotZeroed` asserts measured-bare-ground publishes `instances: 0` with no warning while an unreadable survey omits the field entirely, publishes `unreadableComponents`, warns, and keeps `components` — and that the two are distinguishable by the field's presence. `check_test_ids.py`: CLEAN, 4800 ids, no dot-prefix collisions. **Deliberate coupling change, called out rather than discovered at build time:** `bInstancesMeasured` defaults false (fail-closed, matching `bMeasured` and `bReachMeasured`), so the two existing hand-built fixtures — `PinWrightGrassSettleMakeBuiltReport` in `TestCaptureLandscapeGrassSettle.cpp` and `PinWrightGrassReachMakeSettledReport` in `TestCaptureGrassFrameReach.cpp` — each gained one line setting it true, with a comment saying why. Both fixtures represent complete measurements, so no assertion was weakened: the settle fixture's `ComponentsAfter=4, InstancesAfter=0, bReachMeasured=false` case still publishes `instances: 0` and still carries NO warning (the new warning branch is gated on `!bInstancesMeasured` and is last in the chain, so it cannot fire there), and the reach fixture's out-of-reach and within-reach assertions read the same response they did before. Every other hand-built report in those files is either unmeasured or has `LandscapeProxies == 0`, both of which return from `MakeGrassBuildInfoObject` before any of this. **NOT verified by this change: nothing was compiled and no test was run** — the orchestrator builds and runs after all agents return, and no editor was live to re-measure the 149-component carpet against the new accessor. The count's correctness rests on the engine source cited above, not on a live reading."
