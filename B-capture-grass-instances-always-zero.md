---
id: B-capture-grass-instances-always-zero
title: "the capture response's grass.instances counts PerInstanceSMData through GetInstanceCount(), which landscape grass never populates — 149 grass components rendering a dense carpet report instances: 0, so the one field that says HOW MUCH grass was built is structurally incapable of reporting any"
status: OPEN
severity: High
category: bug
tags: [render, capture_open_level, landscape, grass, vegetation, verification-field, structurally-constant-field, hism, per-instance-render-data, silent-false-negative, getinstancecount]
encounters: 1
lastSeen: 2026-08-30T16:00:00+05:00
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
