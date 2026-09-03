---
id: B-landscape-set-material-stale-mics
title: "landscape.set_material never rebuilds the per-component landscape MICs, so re-assigning the material the landscape already holds is a silent no-op and NO material edit ever reaches the screen"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, material, silent-noop, misleading-success, shader-map, mic, post-edit-change]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `landscape.set_material` reports success while the component MICs stay stale

`landscape.set_material` returns `{"success": true, "message": "Landscape material set"}`
unconditionally. When the `materialPath` resolves to the material the landscape **already
holds**, that assignment is a pointer self-assign and the verb changes nothing — but the
response is indistinguishable from a real assignment.

The consequence is much larger than the assignment itself. The per-component landscape
material instances (`ULandscapeComponent`'s MICs, cached in
`ALandscapeProxy::MaterialInstanceConstantMap`) keep their **stale shader map**, so **no**
edit to the landscape material — topology *or* constant-value — ever reaches the screen.
The base `UMaterial` is fine: `material.authoring.compile_material` runs
`PreEditChange(nullptr)` + `PostEditChange()`, which regenerates the material's `StateId`
and therefore its DDC key. It is the component MICs that are never rebuilt, and
`ALandscapeProxy::UpdateAllComponentMaterialInstances` is **not exposed to Python** on
UE 5.8 (see below), so there is no user-side way to force the rebuild.

This cost multiple sessions and propagated a wrong recipe ("edit values, then
`compile_material`, then `set_material`") into working notes before the round-trip was
found.

## Measured evidence

One fixed camera, foam coverage on the landscape material, UE 5.8, `EAContentExamples58`:

| Step | Foam coverage |
|---|---|
| baseline | 8.85% |
| edit parameter values + `material.authoring.compile_material` | 8.91% (no change) |
| + `landscape.set_material` with the **same** material | 8.91% (still no change) |
| + full round-trip: assign a **different** material, then assign the intended one back | **2.39%** (expected result) |

## Root cause (source-verified 2026-08-13 against HEAD + `C:\UE_5.8`)

`Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp:896-897`:

```cpp
Landscape->LandscapeMaterial = Mat;
Landscape->PostEditChange();
```

Two defects sit in those two lines:

1. **No same-pointer detection.** `Mat == Landscape->LandscapeMaterial` is never checked, so
   a self-assign is reported as a successful assignment.
2. **The bare `PostEditChange()` misses the material branch entirely.**
   `UObject::PostEditChange()` builds an *empty* `FPropertyChangedEvent`
   (`Property == nullptr`, `MemberProperty == nullptr`). Both
   `ALandscapeProxy::PostEditChangeProperty` (`LandscapeEdit.cpp:5940`) and
   `ALandscape::PostEditChangeProperty` (`:6598`) open with
   `MemberPropertyName = PropertyChangedEvent.MemberProperty ? ...GetFName() : NAME_None`
   and dispatch on it. The branch that actually repairs the components —
   `MemberPropertyName == GET_MEMBER_NAME_CHECKED(ALandscapeProxy, LandscapeMaterial)`
   (`LandscapeEdit.cpp:6143` proxy / `:6629` ALandscape), which clears
   `MaterialInstanceConstantMap`, drops the combination-MIC parents, and calls
   `UpdateAllComponentMaterialInstances()` (`:6175` / `:6792`) — therefore **never runs**.
   Neither override has a null-property catch-all. So the MIC rebuild is skipped for *every*
   assignment through this verb, not only the same-pointer one.

**The engine already ships the correct setter.** `LandscapeMaterial` is declared
`UPROPERTY(EditAnywhere, BlueprintSetter=EditorSetLandscapeMaterial, ...)`
(`LandscapeProxy.h:603`), and `ALandscapeProxy::EditorSetLandscapeMaterial`
(`UFUNCTION(BlueprintSetter)`, `LandscapeProxy.h:1020`, impl
`LandscapeBlueprintSupport.cpp:98`) does exactly what the handler does *not*:

```cpp
LandscapeMaterial = NewLandscapeMaterial;
FPropertyChangedEvent PropertyChangedEvent(FindFieldChecked<FProperty>(GetClass(), FName("LandscapeMaterial")));
PostEditChangeProperty(PropertyChangedEvent);
```

The direct rebuild entry points are `ALandscapeProxy::UpdateAllComponentMaterialInstances`
(`LandscapeProxy.h:1414`, `LANDSCAPE_API`) and
`ULandscapeInfo::UpdateAllComponentMaterialInstances` (`LandscapeInfo.h:415`, all proxies).
Both are plain C++ with **no `UFUNCTION`**, so they are reachable from the handler but
invisible to `python.execute` — which is why the caller has no workaround short of the
round-trip.

**Workaround (current, unblocks users today):** round-trip the assignment —
`landscape.set_material` with a *different* material, then `landscape.set_material` with the
intended one. Verified to produce the expected render (2.39% above).

**Fix:**
1. Route the write through `ALandscapeProxy::EditorSetLandscapeMaterial(Mat)` (or replicate
   it: assign, then build a real `FPropertyChangedEvent` naming `LandscapeMaterial` and call
   `PostEditChangeProperty`), so the engine's own MIC-rebuild branch runs. This alone fixes
   the "material edits never reach the screen" symptom for genuine assignments.
2. Detect the same-pointer case explicitly and **force** the MIC rebuild anyway
   (`UpdateAllComponentMaterialInstances(/*bInInvalidateCombinationMaterials=*/true)` or the
   `ULandscapeInfo` variant for all proxies) rather than silently succeeding — "re-assign the
   same material to refresh it" is the natural caller intent and must work.
3. Report it: echo `previousMaterialPath`, a `changed` / `samePointer` boolean, and the number
   of components whose MICs were rebuilt, so a caller can tell a real assignment from a no-op.
   The current response echoes only the requested `materialPath`.
4. Consider a dedicated `landscape.refresh_material` verb (or a `forceRebuild` flag) since
   `UpdateAllComponentMaterialInstances` is unreachable from Python — there is currently no
   supported way to ask for the rebuild on its own.

severity rationale: impact=silent-false-success on a normal path (the verb reports the
assignment done, and the far larger consequence is that every landscape-material edit
silently fails to render, which reads as "my material edits do nothing") x reach=common
(landscape material iteration; it also invalidates the readback of every
`material.authoring.*` edit against a landscape) -> High.

## Related

- `B-create-procedural-terrain-paints-nothing` — sibling landscape silent-false-success; its
  documented recovery path routes through `landscape.set_material`, so a caller following it
  hits this defect next.
- `B-configure-layer-blend-wrong-nodes` — the other half of that recovery path; it does not
  create a `LandscapeLayerBlend` at all.
- `E-landscape-edit-extent-error-not-diagnostic` records a session that already retried
  `landscape.edit` *after* a `landscape.set_material`, consistent with this stale-MIC shape.

## History
- `#1-initial-repro` `OPEN` reporter — "`landscape.set_material` assigns the material pointer the landscape already holds (no same-pointer check) and reports `success:true`, so the per-component landscape MICs keep their stale shader map and NO material edit — topology or constant-value — reaches the screen. The base material is fine after `material.authoring.compile_material` (`PostEditChange` regenerates its StateId -> new DDC key); it is the component MICs that are stale, and `UpdateAllComponentMaterialInstances` is not exposed to Python on UE 5.8. Measured, one fixed camera, foam coverage: baseline 8.85%; edit values + compile_material -> 8.91% (no change); + `landscape.set_material` with the same material -> still 8.91%; + full round-trip (assign a DIFFERENT material, then the intended one back) -> 2.39%, the expected result. Source-verified at HEAD: `LandscapeHandler.cpp:896-897` does `LandscapeMaterial = Mat; PostEditChange();` — the bare `PostEditChange()` builds an empty `FPropertyChangedEvent`, so `MemberPropertyName` is `NAME_None` in `ALandscapeProxy::PostEditChangeProperty` (`LandscapeEdit.cpp:5940`) and `ALandscape::PostEditChangeProperty` (`:6598`), and the `GET_MEMBER_NAME_CHECKED(ALandscapeProxy, LandscapeMaterial)` branch (`:6143` / `:6629`) that clears `MaterialInstanceConstantMap` and calls `UpdateAllComponentMaterialInstances()` (`:6175` / `:6792`) never runs — for ANY assignment through this verb, not just the same-pointer one; neither override has a null-property catch-all. The engine's own setter `ALandscapeProxy::EditorSetLandscapeMaterial` (`UFUNCTION(BlueprintSetter)`, `LandscapeProxy.h:1020`, impl `LandscapeBlueprintSupport.cpp:98`) does it correctly by constructing a real `FPropertyChangedEvent` naming `LandscapeMaterial`. Fix: route through that setter (or replicate it), force the MIC rebuild on the same-pointer case instead of silently succeeding, and echo `previousMaterialPath` / `changed` / rebuilt-component count so a no-op is visible. Cost multiple sessions and propagated a wrong recipe into working notes; workaround is the round-trip."
- `#2-fix` `IN-REVIEW` developer — Added `SetLandscapeMaterialAndNotify(ALandscape*, UMaterialInterface*)` in `Source/PinWright/Private/Handlers/Environment/LandscapeHandler.cpp`: it assigns `LandscapeMaterial`, then builds a real `FPropertyChangedEvent` from `FindFProperty<FProperty>(ALandscapeProxy::StaticClass(), GET_MEMBER_NAME_CHECKED(ALandscapeProxy, LandscapeMaterial))` and calls `PostEditChangeProperty`, so the engine's `LandscapeMaterial` branch runs and `MaterialInstanceConstantMap` is emptied + `UpdateAllComponentMaterialInstances()` called. **Correction to the ticket's proposed fix:** routing through `ALandscapeProxy::EditorSetLandscapeMaterial` is NOT possible from this module — `ALandscapeProxy` is `UCLASS(MinimalAPI)` and the setter carries no `LANDSCAPE_API`, so only its type info is exported and a direct call would not link. Reconstructing the event is the linkable equivalent, because `PostEditChangeProperty` is virtual and dispatches through the vtable. `ALandscape::PostEditChangeProperty` chains to `Super::`, so one call runs both the proxy branch (`LandscapeEdit.cpp:6143`) and the ALandscape branch (`:6629`). Applied at both call sites that assigned `LandscapeMaterial`: `landscape.set_material` and the `materialPath` branch of `landscape.create` (which had the identical bare `PostEditChange()`). Fix-list item 2 (same-pointer force rebuild) needs no separate handling — the rebuild is driven by the property-named notification, not by the pointer differing, so re-assigning the same material now rebuilds the MICs; the response gained `previousMaterialPath` and `changed` (fix-list item 3) plus a distinct success message so a refresh is visible. Fix-list item 4 (a dedicated `landscape.refresh_material` verb / `forceRebuild` flag) NOT done — with same-pointer re-assignment working it is no longer needed to reach the rebuild; leaving it out avoids adding a verb for a capability `set_material` now covers. Test: `Source/PinWright/Private/Tests/Core/LandscapeSetMaterialRebuildsMicsTest.cpp` — `PinWright.landscape.set_material.RebuildsComponentMaterialInstances` builds a real landscape through the production `landscape.create` handler, seeds a sentinel entry into `ALandscapeProxy::MaterialInstanceConstantMap`, drives the production `landscape.set_material` handler, and asserts the sentinel is gone; emptying that map is the exact engine work the empty event skipped, so the test fails pre-fix and needs no external content beyond `/Engine/EngineMaterials/WorldGridMaterial`. Docs: `Docs/wiki-src/landscape.md` and `Docs/wiki-src/level-blockout.md` no longer teach the round-trip as the supported route — both now document the single `set_material` call and keep the round-trip noted as a workaround for older builds, with the pointer-self-assign explanation corrected. Did not compile/run (later integration phase); the acceptance test is that the round-trip workaround becomes unnecessary and the foam-coverage metric reaches 2.39% from a single `set_material`.
