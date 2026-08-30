---
id: B-grass-varieties-edit-does-not-reach-renderer
title: "Editing a ULandscapeGrassType's GrassVarieties and saving reaches the renderer not at all (meanLuminance 0.4444 -> 0.4439 at a fixed pose), and the two flush paths a caller reaches for do not exist — grass.FlushCacheAll is EXEC_FAILED and unreal.Landscape.flush_grass_components is an AttributeError; only a raw system.console_command grass.FlushCache works"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, grass, vegetation, grass-type, silent-noop, derived-state, consumer-refresh, flush, misleading-success, material-consumers-precedent, flush-grass-maps, destructive-workaround, session-state, acceptance-test-gap, returned-by-tester, regression, grass-map-deletion, live-verified]
encounters: 3
lastSeen: 2026-08-30T16:55:00+03:00
---

# The edit lands on the asset and never lands on the screen

Editing a `ULandscapeGrassType`'s `GrassVarieties` and saving it produces a frame that is
indistinguishable from the one before the edit. Measured at a fixed pose:

| step | `meanLuminance` |
|---|---|
| baseline | 0.4444 |
| edit `GrassVarieties`, save | 0.4439 |
| `system.console_command {command: "grass.FlushCache"}` | **0.4287** |

0.4444 -> 0.4439 is not a small effect; it is no effect. The flush is what makes the edit visible,
and every verb in the chain reported success without it.

## The two flush paths a caller would reach for do not exist

Both were tried live, both failed, and neither failure names the thing that does work:

- `grass.FlushCacheAll` -> `EXEC_FAILED`. Correct: the engine registers `grass.FlushCache` and
  `grass.FlushCachePIE` and nothing else in that family —
  `C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:3474-3483` are the two
  `FAutoConsoleCommand` registrations, backed by `FlushGrass` (`:3441-3447`) and `FlushGrassPIE`
  (`:3449-3455`). There is no `All` variant to find.
- `unreal.Landscape.flush_grass_components` -> `AttributeError`. Also correct:
  `ALandscapeProxy::FlushGrassComponents` is declared `LANDSCAPE_API` but **not** `UFUNCTION`
  (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Classes/LandscapeProxy.h:1165`, doc comment at
  `:1162-1164`), so it is C++-only and never reaches Python. The same is true of the subsystem
  entry point, `ULandscapeSubsystem::RegenerateGrass`
  (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Public/LandscapeSubsystem.h:120`, doc at `:114-119`)
  — `LANDSCAPE_API`, no `UFUNCTION`.

So the only working route is a raw console string through `system.console_command`, which a caller
finds by reading engine source. That is the Medium band's "source dive" as the *only* path to
making a supported edit take effect.

## PinWright has no grass flush at all

Grepped across `Plugins/PinWright/Source/PinWright/` (all `.cpp` and `.h`): zero hits for
`FlushGrass`, `RegenerateGrass`, or `grass.FlushCache`. Nothing in the plugin flushes, rebuilds or
even mentions the grass cache, so no verb that edits a grass type could be routing through one.

## This codebase already has the pattern, applied to materials and not to grass

The fix shape is not novel here — it is an established house pattern that this path did not get.

`PinWright::MaterialConsumers::ApplyMasterMaterialEdit(Material, Report)`
(`Source/PinWright/Private/Handlers/Material/MaterialLandscapeConsumers.h:316-322`) is the "this
master edit is finished" call: it notifies the material change and then rebuilds the landscape
consumers, *measuring* what it refreshed into an `FConsumerRefreshReport`. Its own comment at
`:311-315` states the rule verbatim:

> Every verb that COMPLETES a unit of work on a master material should call this instead of a bare
> `PostEditChange()`. [...] having one name for it is what stops the next verb from shipping the
> bare pair again — which is exactly how this defect survived its first fix.

It is called from `material.authoring.configure_layer_blend`
(`MaterialAuthoringHandler.cpp:3519`, with `AddKnownUnrefreshedConsumers` at `:3520` and the
measured report published at `:3591`) and from `material.authoring.compile_material`
(`MaterialAuthoringHandler.cpp:3628`). The header's own preamble
(`MaterialLandscapeConsumers.h:26-40`) spells out why the bare `PostEditChange()` is insufficient
for landscape: landscape keeps a private cache the engine's material-change plumbing cannot see.

**A `ULandscapeGrassType` is the same situation with a different cache.** The grass instances are
built and held per landscape component; editing the type asset does not invalidate them, so the
edit measures as a no-op while every verb reports success. The material path got a named
"completes a unit of work" call and a measured refresh report; the grass path got neither.

The board records what this cost on the material side: `B-compile-material-landscape-consumers-stale`
(DONE, High) — a graph edit that "measures its own work as a no-op while every verb in the chain
reports success", where the destructive control moved a frame 1.81% (the noise floor) before the
round-trip and 17.19% after. `B-landscape-set-material-stale-mics` (IN-REVIEW, High) records the
same shape costing "multiple sessions" and propagating a wrong recipe into working notes. This
ticket is that defect class, third occurrence, on grass types.

## Fix

Either half is acceptable; the first is preferred because it matches the precedent.

1. **Make the edit path flush, the way the material path already refreshes.** A named
   `ApplyGrassTypeEdit(GrassType, Report)` beside `ApplyMasterMaterialEdit`, called by every verb
   that completes a unit of work on a `ULandscapeGrassType`, driving
   `ULandscapeSubsystem::RegenerateGrass(bInFlushGrass=true, bInForceSync=true, ...)`
   (`LandscapeSubsystem.h:120`; the implementation's flush branch is
   `LandscapeSubsystem.cpp:624-627` and its camera-locations branch `:633-636`) and publishing an
   `FConsumerRefreshReport` so the response *measures* the rebuild instead of asserting it.
2. **A typed flush verb** — `landscape.flush_grass` — so the capability at least exists by name and
   is discoverable, rather than living in an engine console string. This is the weaker fix on its
   own: it leaves every edit verb still reporting success for a change nothing has applied, which
   is the actual defect.

Whichever lands, the response must name what was refreshed. The precedent's
`AddKnownUnrefreshedConsumers` (`MaterialLandscapeConsumers.h:326-337`) exists precisely so a
coverage number is not read as broader than it is.

**Workaround:** after any grass-type edit, `system.console_command {command: "grass.Enable 0"}` then
`{command: "grass.Enable 1"}` — measured effective, rebuilds in ~45 s with the edited settings live.

> **DO NOT use `grass.FlushCache` as the workaround.** It was the original recommendation on this
> line and it is destructive in the editor: measured 922,280 -> 56,080 grass instances with no
> recovery short of an editor restart. The prose above and the `0.4439 -> 0.4287` measurement in the
> table are the original filing and are left as they were; the recommendation is corrected here and
> the evidence is in `#3`, which also flags a line in the IN-REVIEW implementation that copies the
> same destructive argument.

## Same shape as

- `B-compile-material-landscape-consumers-stale` (DONE, High) — the precedent this ticket argues
  from. Same class, materials.
- `B-landscape-set-material-stale-mics` (IN-REVIEW, High) — same class, landscape material
  assignment.
- `B-anim-compile-stale-dirty` — same class, animation compile.

Not a duplicate of any of them: those three are about `UMaterial` / `UMaterialInstanceConstant`
shader maps and landscape combination MICs. This is the landscape *grass instance* cache, a
different cache reached by a different verb family, and no fix to a material verb touches it.

Distinct from `B-create-grass-type-addzeroed-never-renders` (DONE), which was about the asset being
created inert. Here the asset is valid and the *edit* to an already-valid asset does not propagate.

## Severity

**High**, by impact class: *silent false-success* — the edit verb returns success, the asset on
disk is correct, and the renderer shows the old grass. The rubric's parenthetical is the exact
failure: "the caller trusts a result that is a lie and builds on it". The named consequence in the
sibling tickets is the one that costs real time — an agent tuning a value concludes it has no
effect and keeps raising it.

**Reach modifier declined.** Grass-type editing is not an every-session path, which by the rubric
would bump this down to Medium. Declined because this is the third occurrence of a defect class
this project has already paid for twice at High
(`B-compile-material-landscape-consumers-stale`, `B-landscape-set-material-stale-mics`), and
because a named, documented fix pattern for exactly this exists in this codebase and was not
applied here — rating the third instance lower than the first two would sort the fix behind work
that is less well understood.

Not rated Critical: nothing is corrupted and the asset itself is written correctly; only the
derived render state is stale.

## History
- `#1-grass-varieties-edit-never-flushes` `OPEN` reporter — Measured live against a running editor
  at a fixed pose: `GrassVarieties` edit + save moved `meanLuminance` 0.4444 -> 0.4439 (i.e. not at
  all); `system.console_command {command: "grass.FlushCache"}` then moved it 0.4439 -> 0.4287.
  `grass.FlushCacheAll` returned `EXEC_FAILED` and `unreal.Landscape.flush_grass_components` raised
  `AttributeError`. Both failures re-derived against UE 5.8 source: only `grass.FlushCache` and
  `grass.FlushCachePIE` are registered (`LandscapeGrass.cpp:3474-3483`, bodies at `:3441-3455`),
  and neither `ALandscapeProxy::FlushGrassComponents` (`LandscapeProxy.h:1165`) nor
  `ULandscapeSubsystem::RegenerateGrass` (`LandscapeSubsystem.h:120`) is a `UFUNCTION`, so neither
  reaches Python. Grepped the whole plugin: zero hits for `FlushGrass` / `RegenerateGrass` /
  `grass.FlushCache`. Precedent cited for the ask:
  `PinWright::MaterialConsumers::ApplyMasterMaterialEdit` (`MaterialLandscapeConsumers.h:316-322`,
  rule stated at `:311-315`), called at `MaterialAuthoringHandler.cpp:3519` and `:3628`.
- `#2-grass-flush-on-reflected-edits` `IN-REVIEW` developer — "Root cause found and it is an engine behaviour change, not just a missing call: UE 5.3's `ULandscapeGrassType::PostEditChangeProperty` flushed the consumers (`Proxy->FlushGrassComponents()`), and from UE 5.4 that body only calls `InvalidateGrassTypeSummary()` + recomputes `StateHash`, neither of which invalidates `ALandscapeProxy::FoliageCache.CachedGrassComps` — a cache keyed on the grass type POINTER and the variety COUNT and on nothing inside an `FGrassVariety`. Added `Handlers/Environment/GrassTypeConsumers.h` (`PinWright::GrassConsumers`) as the grass twin of `MaterialLandscapeConsumers.h`: `RefreshGrassConsumers` finds every editor-world landscape holding cached grass keyed on the type (or whose components declare it), calls `ALandscapeProxy::FlushGrassComponents(nullptr, bFlushGrassMaps=true)` — the C++ entry point `grass.FlushCache` itself calls, LANDSCAPE_API on 5.3-5.8 — then `ULandscapeSubsystem::RegenerateGrass(false, true)`, and MEASURES the result into an `FConsumerRefreshReport` by snapshotting the keyed cache entries and their HISM pointers before and after. `ApplyGrassTypeEdit` is the named pairing beside `ApplyMasterMaterialEdit`. Wired it at the seam where the edit actually happens: `NotifyReflectedPropertyChanged` in `UtilityPropertyHandler.cpp`, beside the existing `PushRenderStateForComponentTarget`, so `property.set`, `property.reset` and every `container.array.*`/`container.map.*` write to a `ULandscapeGrassType` flushes; `property.set` also publishes the measured `consumerRefresh` block (gated so no other target class pays anything but a failed Cast). `landscape.create_grass_type` deliberately NOT wired — a just-created asset no material references has no consumers — and the header says so. Added the typed verb `landscape.flush_grass` (new file `Handlers/Environment/LandscapeGrassFlushHandler.cpp`, so the two agents in `LandscapeHandler.cpp` were not touched) for edits made outside the plugin. MEASUREMENT HONESTY: the regression tests measure the per-proxy grass CACHE, not luminance — `PinWright.property.set.GrassTypeEditFlushesGrassCache` and `PinWright.landscape.flush_grass.StaleGrassCacheIsInvalidated` build a 1x1 landscape, seed one `FoliageCache.CachedGrassComps` entry keyed on a sandbox grass type, and assert it is gone afterwards plus that `consumerRefresh` reports measured/consumersFound>=1/consumersRefreshed>=1 and NAMES the landscape in `refreshed[]`. Pre-fix the entry survives and no block is emitted. Nothing here proves a pixel moved: a headless host cannot bake grass maps and place a camera, so the 0.4444/0.4287 luminance claim is NOT re-verified by this change and still wants a live-editor pass. On the two non-existent flush paths: grepped the whole plugin tree, `Docs/`, `Saved/PinWright/wiki/` and `asset-dumps/` — `grass.FlushCacheAll` and `unreal.Landscape.flush_grass_components` appear NOWHERE except this ticket, so there was no wrong text in this tree to correct; instead `Docs/wiki-src/landscape.md` now states outright that neither exists and why (not a UFUNCTION / not a registered command), and documents `landscape.flush_grass` plus the `grass.FlushCache` console fallback. One bullet added to `Docs/rpc-design.md` §5b recording the third occurrence and the engine-version trap. Not compiled or run — orchestrator builds. Known nit: the `landscape` namespace page was already ~23.9 KB (over the ~20 KB soft guideline) before this change and is now ~25.5 KB."
- `#3-flushcache-is-destructive-and-the-fix-copied-it` `IN-REVIEW` reporter — **Second encounter, and it
  reports a hazard in this ticket's own recommended workaround and in one line of `#2`'s
  implementation. Status deliberately NOT changed: I was not asked to verify the fix and am not acting
  as tester. This is evidence for whoever does.**
  **Measured, live editor, `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8), method
  in `Docs/map/vegetation-performance.md` § *Measurement hazards* item 7:**
  `system.console_command {command: "grass.FlushCache"}` took the level from **922,280 grass instances
  to 56,080** and **the grass did not come back** — not through camera moves, not through
  `set_grass_enabled` false/true, not after four minutes idle. **Only an editor restart recovered it.**
  On a project whose editor is shared by five agents, "restart to recover" is not a workaround; it
  destroys every other agent's unsaved work.
  **The destructive ingredient is the second argument, and the engine names it.**
  `ALandscapeProxy::FlushGrassComponents(const TSet<ULandscapeComponent*>* OnlyForComponents = nullptr,
  bool bFlushGrassMaps = true)` — `C:/UE_5.8/Engine/Source/Runtime/Landscape/Classes/LandscapeProxy.h:1165`,
  doc comment `:1162-1164`: *"bFlushGrassMaps will delete the grass data / density maps on the components
  as well, **but only in editor mode**, and only if the grass maps are renderable (i.e. they can be
  regenerated)."* The two console commands differ in exactly that argument:
  `grass.FlushCache` -> `FlushGrass` -> `Landscape->FlushGrassComponents();` (defaulted **true**),
  `C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:3441-3447`, registered `:3474-3478`;
  `grass.FlushCachePIE` -> `FlushGrassComponents(nullptr, false)`, `:3449-3455`, registered `:3480-3484`.
  Despite its name, **the PIE variant is the non-map-deleting one**, and the editor-only clause in the
  doc comment is why the destructive path is reachable only where we hit it.
  **The engine's own regeneration entry point passes `false`, and that is the deciding citation.**
  `ULandscapeSubsystem::RemoveGrassInstances` (`Private/LandscapeSubsystem.cpp:602-611`) ends at `:609`
  with `Proxy->FlushGrassComponents(ComponentsToRemoveGrassInstances, /*bFlushGrassMaps = */false);` —
  argument name spelled out in a comment — and `ULandscapeSubsystem::RegenerateGrass(bInFlushGrass,
  bInForceSync, ...)` (declared `Public/LandscapeSubsystem.h:120`, implemented
  `Private/LandscapeSubsystem.cpp:613`) reaches it at `LandscapeSubsystem.cpp:624-627` when
  `bInFlushGrass` is true, then runs `UpdateGrass` at `:629+`. So `RegenerateGrass(true, true)` — the
  exact call this ticket's own **Fix #1** asked for — flushes instances and **never deletes a grass
  map**. It is the safe shape and it was already written down here.
  **What `#2` implemented instead, flagged for the tester rather than judged.** `#2` states it calls
  `ALandscapeProxy::FlushGrassComponents(nullptr, bFlushGrassMaps=true)` — describing it as *"the C++
  entry point `grass.FlushCache` itself calls"*, which is exactly right and is the problem — followed by
  `RegenerateGrass(false, true)`, i.e. `bInFlushGrass=false`, so nothing after the deletion re-drives a
  map rebuild. That is the destructive combination, wired onto `NotifyReflectedPropertyChanged` (so every
  `property.set` / `property.reset` / `container.*` write to a `ULandscapeGrassType`) and exposed as the
  new typed verb `landscape.flush_grass`. **The acceptance tests `#2` added cannot detect it**, and `#2`
  says so in its own words — they assert the `FoliageCache.CachedGrassComps` entry is *gone* and that
  `consumerRefresh` reports what it refreshed, and it notes *"Nothing here proves a pixel moved"*. A
  cache-entry-disappeared assertion passes identically whether the grass rebuilds in 45 seconds or never
  returns until restart. **Suggested before verification, not applied here** (I have not touched plugin
  source): pass `bFlushGrassMaps=false`, or drop the direct `FlushGrassComponents` call and use
  `RegenerateGrass(/*bInFlushGrass=*/true, /*bInForceSync=*/true)` which does it correctly on its own;
  and add a live-editor acceptance step that the grass instance count **returns**, not merely that the
  cache key vanished. The `landscape.flush_grass` verb should also state in its summary which of the two
  behaviours it has.
  **The safe console route, with its mechanism**, replacing the workaround line in the body:
  `grass.Enable 0` then `grass.Enable 1` — measured to drop and rebuild the grass components in ~45 s
  with the edited settings live, no restart. `GGrassEnable` is declared at `LandscapeGrass.cpp:144-148`
  and gates `ALandscapeProxy::ShouldGenerateGrass()` at `:1292`; nothing on that path deletes a grass
  map, which is why it is recoverable and `FlushCache` is not. Also confirmed against the engine and
  recorded so nobody re-derives it: **no `BuildGrassMaps` console command exists** and `ALandscapeProxy`
  reflects only `get_grass_enabled` / `set_grass_enabled` to Python, so there is no reflected rebuild
  trigger either. **Body edit made and declared:** the `**Workaround:**` line now recommends
  `grass.Enable 0/1` and carries a warning block; the original prose and the `0.4439 -> 0.4287`
  measurement are left untouched beside it. **Severity left at High, not raised.** Critical is declined
  on the rubric as written — nothing crashes and no asset data is corrupted or lost; the grass maps are
  derived render state and a restart restores everything. Recorded against that: if `#2` ships as
  described, the destruction moves from a console string a caller opts into onto **every reflected
  property write to a grass type**, which is the point at which a re-rating should be revisited.
  `encounters` 1 -> 2.

- `#4-returned-flush-grass-deletes-the-grass-maps` `OPEN` tester — **Returned to OPEN. Acting as tester on explicit instruction. The fix does not work: `landscape.flush_grass` destroys the grass carpet and it does not come back — the destruction `#3` predicted, now reachable through a typed verb and through every reflected write to a grass type.** **Measured live, editor build 13:32 (`Plugins/PinWright/Binaries/Win64/UnrealEditor-PinWright.dll` mtime 13:32; plugin commit `d8f1bc32` "Work 33 Critical+High board tickets, waves 2 and 3"), host `EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`, one landscape actor `Terrain`, grass type `/Game/VegetationTest/GrassTypes/LGT_VegTest_Meadow`.** Camera pinned at `(-19000, -15000, 338)` `pitch -3 / yaw 325`, fov 50; exposure PINNED at `ev100 -0.5` on every comparison shot (one `{mode:"auto"}` probe first returned `ev100Equivalent -0.5000003`, and that value was passed to all five) so no pair is comparable by accident; every capture 512x512, one constant size, never varied mid-run. | step | `grass.components` | `meanLuminance` | PNG bytes | what the frame shows | |---|---|---|---|---| | before | **136** | 0.6855 | 557,315 | full waist-high meadow carpet, rust seed heads, yellow buttercups | | `landscape.flush_grass`, very next capture | **0** | 0.7222 | 329,008 | **bare landscape material, no grass at all** | | + ~1 min, same pose | **0** | 0.7221 | 327,824 | still bare | | + ~3 min, same pose | **0** | 0.7222 | 326,968 | still bare | | after `grass.Enable 0` then `grass.Enable 1`, + ~75 s | **0** | 0.7224 | 323,805 | **still bare — the documented workaround does not recover this either** | **I looked at the pixels, not only at the numbers.** `PW_FG_before.png` is a dense meadow; every frame after the flush is flat pale-green landscape material with the rock line, the birch knoll, the treeline and the sky otherwise unchanged. The direction confirms it: grass darkens the frame, so luminance rising 0.6855 -> 0.7222 while the PNG nearly halves is the grass leaving — the same signature `B-capture-open-level-pose-params-photograph-stale-grass` records in the opposite direction. Captures kept at `Saved/Screenshots/OpenLevel/PW_FG_{before,after,after2,after3,after_toggle}.png`. **The verb reported a clean, measured success while doing it.** Response: `{"success":true, "consumerRefresh":{"measured":true,"consumersFound":1,"consumersRefreshed":1,"consumersWithNothingToRefresh":0,"subObjectsRefreshed":136,"complete":true,"refreshed":["Terrain"]}}`, message *"Grass refreshed on 1 of 1 landscape(s)"*. This is `#2`'s own stated measurement-honesty limit arriving in the field: a cache-entry-disappeared assertion passes identically whether the grass rebuilds or never returns, and `subObjectsRefreshed: 136` is a count of what was destroyed reported as coverage. **Root cause, re-derived at plugin HEAD — `#3`'s flagged line shipped unchanged.** `Source/PinWright/Private/Handlers/Environment/GrassTypeConsumers.h:233` is `Proxy->FlushGrassComponents(/*OnlyForComponents=*/nullptr, /*bFlushGrassMaps=*/true);` and `:239` is `Subsystem->RegenerateGrass(/*bInFlushGrass=*/false, /*bInForceSync=*/true);`. `git log -1 -- GrassTypeConsumers.h` returns `d8f1bc32`, so the file is exactly the build under test and has not moved since. The header states the choice deliberately at `:158-169` and gives its reason at `:163-165` — *"the map-clearing branch is WITH_EDITOR-only and the maps rebuild from the landscape material (5.8 LandscapeGrass.cpp:2723-2737)"*. **That premise is false, and it is the whole defect.** **Why the maps do not rebuild, from engine source.** `ALandscapeProxy::FlushGrassComponents` (`C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeGrass.cpp:2658`) takes its `bFlushGrassMaps` branch at `:2726`, whose body `:2728-2735` calls `ULandscapeComponent::RemoveGrassMap()` on every landscape component. `RemoveGrassMap` (`:1233-1239`) is one statement — `GrassData = MakeShared<FLandscapeComponentGrassData>();` — which replaces the component's density data with a freshly allocated **empty** one; its own comment calls it "a thread safe replacement of the existing grassdata with a newly allocated empty (invalid) one". Nothing on that path schedules a rebuild. The one in-editor path that would regenerate it is `FLandscapeGrassMapsBuilder`'s streaming update, and that is gated off by default: `GGrassMapUseRuntimeGeneration = 0` (`LandscapeGrassMapsBuilder.cpp:43-47`, *"When enabled the grass density maps are not serialized and are built on the fly at runtime"*), tested at `:787-790` and `:873`. With runtime generation off the maps come from the package and are never rebuilt in-session — which is exactly why `grass.Enable 0/1` recovers a plain instance drop and cannot recover this one. **`RegenerateGrass(bInFlushGrass=false, ...)` cannot repair it either, and the header says why in its own words** at `:167-169`: `RegenerateGrass`'s flush branch goes through `RemoveGrassInstances()`, which passes `bFlushGrassMaps=false`. Instances are not maps. Each of the three post-flush captures independently ran the plugin's own settle helper (`Source/PinWright/Private/Handlers/Render/LandscapeGrassSettle.cpp:115`, `RegenerateGrass(/*bInFlushGrass=*/false, /*bInForceSync=*/true, {cameraLocation})`) with the real camera and still measured `components: 0`, so a camera-located regenerate does not bring it back. **Two concrete fixes, either sufficient, in preference order.** (a) Pass `bFlushGrassMaps=false` at `GrassTypeConsumers.h:233`. That is what the engine's own regeneration entry point does — `ULandscapeSubsystem::RemoveGrassInstances` ends at `C:/UE_5.8/Engine/Source/Runtime/Landscape/Private/LandscapeSubsystem.cpp:609` with the argument name spelled out in a comment — and what `grass.FlushCachePIE` does, and it is sufficient to invalidate the cached instances, which is all this ticket ever needed. (b) If the maps genuinely must be dropped, rebuild them inside the same call: `ULandscapeSubsystem::BuildGrassMaps(UE::Landscape::EBuildFlags)` is `LANDSCAPE_API`, declared at `C:/UE_5.8/Engine/Source/Runtime/Landscape/Public/LandscapeSubsystem.h:139` (*"Synchronously build grass maps for all components"*), implemented at `LandscapeSubsystem.cpp:1048`, with a world-walking free function `UE::Landscape::BuildGrassMaps` at `LandscapeSubsystem.cpp:239-251`. `RefreshGrassConsumers` calls neither. It is **not** a `UFUNCTION`, so it is unreachable from Python — a C++ call from this header is the only route, which is itself an argument for the plugin owning the rebuild rather than telling callers to flush. **The acceptance tests must change too, and `#2` already named the gap in its own words.** `PinWright.landscape.flush_grass.StaleGrassCacheIsInvalidated` and `PinWright.property.set.GrassTypeEditFlushesGrassCache` assert the cache entry is *gone*. A correct fix has to assert something *came back*; the simplest headless-reachable form is that `ULandscapeComponent::GrassData->HasValidData()` is still true after the refresh — `bFlushGrassMaps=true` makes it false, `false` leaves it untouched. **The blast radius is larger than the verb, which is the escalation `#3` said would justify revisiting the rating.** `RefreshGrassConsumers` is also wired into `NotifyReflectedPropertyChanged` (`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:386`, helper at `:307-325`), so **every `property.set`, `property.reset` and `container.array.*` write to a `ULandscapeGrassType` now deletes that landscape's grass maps for the session.** The route a caller previously had to opt into by typing a console string is now the default path for editing a grass type at all — while the verb's registered summary (`Source/PinWright/Private/Handlers/Environment/LandscapeGrassFlushHandler.cpp:38-49`) promises the opposite: *"Invalidate and **rebuild** the grass instances"*. **Severity left at High; Critical considered and declined.** The rubric's Critical band is "editor crash, or a write that corrupts or loses asset data". Nothing crashed, and no `.uasset` is touched — grass maps are derived render state that a level reload rebuilds. What is lost is a session, and in an editor shared by several agents that is other people's unsaved work, which is the real argument for Critical; I decline it because the rubric names asset data and this is not asset data. The honest escalation is instead that the destruction has moved onto the **normal** editing path from behind an opt-in console string, and that the response now *measures and publishes* a refresh that did not happen — squarely the High band's "the caller trusts a result that is a lie and builds on it". **State left behind by this test, declared.** The grass for `/Game/Maps/PW_VegetationTest` is gone in that editor until it restarts; I did not restart it (out of scope, and other agents share it). `grass.Enable` was toggled 0 then 1 and is back at its original value, but is now pinned at `ECVF_SetByConsole` for the session. The level was not saved. The viewport camera was restored to the pose it held before the test. `encounters` 2 -> 3.

- `#5-flush-is-now-instances-only-and-the-test-can-tell` `IN-REVIEW` developer — "**Fixed both surfaces by passing `bFlushGrassMaps=false`, and closed the acceptance gap that let the destructive version ship.** **Bisect window, stated first because it is what a bisector needs:** the destructive refresh was committed AND PUSHED as plugin commit `d8f1bc32` ('Work 33 Critical+High board tickets, waves 2 and 3'). Every build from `d8f1bc32` up to this change deletes a landscape's grass density maps on `landscape.flush_grass` AND on every `property.set` / `property.reset` / `container.array.*` / `container.map.*` write to a `ULandscapeGrassType`. **Re-derived from `C:/UE_5.8/Engine/Source` rather than taking either #3 or #4 on trust, and one of #4's mechanism claims does not hold.** `ALandscapeProxy::FlushGrassComponents` (`Private/LandscapeGrass.cpp:2658`) has two branches; the no-filter branch this code takes runs `FoliageComponents.Empty()`, `FoliageCache.ClearCache()` and the destruction of every owned and attached HISM **unconditionally** at `:2691-2721`, and `FCachedLandscapeFoliage::ClearCache()` is one statement — `CachedGrassComps.Empty()` (`Classes/LandscapeProxy.h:386-389`). So the pointer-and-count-keyed cache entries #2 diagnosed as never invalidated are gone with `bFlushGrassMaps=false`: the destructive argument was buying this path nothing. What it gated is only the extra `WITH_EDITOR` block at `:2726-2736`, `ULandscapeComponent::RemoveGrassMap()` per component, which is `GrassData = MakeShared<FLandscapeComponentGrassData>()` (`:1233-1239`) — a default-constructed struct whose `NumElements` is `UnknownNumElements` (-1), so `HasValidData()` (`:1653-1659`) goes false. **What is lost, and whether it is regenerable.** Lost is the per-component grass density/weight map — `HeightWeightData`, `WeightOffsets`, `HeightMipData`, `GenerationHash` — the GPU rasterisation of the landscape material's grass output that the HISM instances are built *from*. It is **serialised into the landscape package** (`Private/Landscape.cpp:1114`, `Ar << GrassData.Get()`), so it is not purely transient: saving the level while it is empty makes the loss durable, which is a stronger claim than #3's and #4's 'a restart restores everything'. It IS regenerable in principle, and #4's stated reason for saying otherwise is wrong — `GGrassMapUseRuntimeGeneration` gates nothing in editor, because `LandscapeGrassMapsBuilder.cpp:787-790` sits under `#if !WITH_EDITOR` and `:873` is the `#else` half of a `WITH_EDITOR` branch; the editor path says outright that it wants all maps built and kept (`:862-865`), and `UpdateTrackedComponents` detects exactly this event — comment at `:466-471`, *'detect if grass data has been cleared by someone manually calling Flush'* — and evicts the component back to `Pending` (`CancelAndEvict`, `:1345-1366`). But that rebuild is amortised (`AmortizedUpdate.ShouldUpdate`, `:460`) and gated on `GGrassEnable && Cameras.Num() > 0 && bAllowStartGrassMapGeneration` plus a free pipeline slot (`:880`), which is consistent with #4 measuring zero recovery over three minutes. So: regenerable, but best-effort, asynchronous, camera-driven, and not reachable synchronously from this plugin. #4's conclusion stands even though its mechanism does not. **THE DECIDING CITATION, and it is why there is no opt-in parameter.** The engine's own grass-map builder, on detecting that a component's grass TYPES changed, comments *'this invalidates foliage instances but not the grass maps'* (`LandscapeGrassMapsBuilder.cpp:500`), collects the component (`:504`) and routes it through `ULandscapeSubsystem::RemoveGrassInstances` (`:579`), which ends at `LandscapeSubsystem.cpp:609` with `FlushGrassComponents(Components, /*bFlushGrassMaps = */false)`. A grass-type edit changes which meshes and densities the instances are built with, never the material's rasterised weight output the map holds — so no edit class needs the destructive variant, and an opt-in for a mode nothing needs would be a knob published as a promise (rpc-design.md §1). #4's option (b), `ULandscapeSubsystem::BuildGrassMaps`, was therefore not taken: there is nothing to rebuild. A caller who genuinely wants every map in the process dropped still has `system.console_command {command: 'grass.FlushCache'}`, where the cost is opted into by name. **Changes.** `Handlers/Environment/GrassTypeConsumers.h` now passes `bFlushGrassMaps=false`, with the argument-by-argument citation replacing the false premise the header stated at its old `:163-165`. Both surfaces share that header, so the typed verb (`LandscapeGrassFlushHandler.cpp`) and the automatic hook (`UtilityPropertyHandler.cpp`, `NotifyReflectedPropertyChanged` -> `RefreshGrassConsumersForNotifiedTarget`) are fixed by the same line — and the hook, which fires with no opt-in at all, is the one that mattered most. **The response now carries the measurement `consumerRefresh` structurally cannot make:** a new `grassMaps` block — `measured`, `componentsHoldingMapsBefore`, `componentsHoldingMapsAfter` — counted across the same consumer set on both sides of the flush INSIDE the call, so no later tick can confuse it. Per the house convention it publishes only measured values (there is no request to name: this path has no destructive mode to ask for), `discarded` is OMITTED when zero so its presence rather than its value is the signal, and when non-zero it arrives with a `warning` string and the verb's success `message` says so too — because a lost map contradicts the sentence beside it. Report threading moved from a bare `FConsumerRefreshReport` to `PinWright::GrassConsumers::FGrassRefreshReport { Consumers, GrassMaps }` so the two measurements cannot be taken apart by a future call site. The registered summary of `landscape.flush_grass` — which #4 correctly flagged as promising the opposite of what it did — now states outright that it is non-destructive and that `grass.FlushCache` is NOT equivalent. **Acceptance tests extended so they FAIL against `d8f1bc32`, which is the real lesson.** `PinWright.property.set.GrassTypeEditFlushesGrassCache` and `PinWright.landscape.flush_grass.StaleGrassCacheIsInvalidated` asserted only that the seeded cache entry was gone, and 'gone' reads identically for an invalidation and for a destruction. Each now also seeds a COMPUTED grass map onto every landscape component before the call (`SeedComputedGrassMapForGrassFlushTest`) and asserts it survives, from two independent readings: engine state (`ULandscapeComponent::GrassData->NumElements >= 0`) and the response's `grassMaps` block. The seed is load-bearing — a freshly built automation landscape has never had a map computed, so without it the assertion would be vacuously true and would pass against the destructive build. On the accessor: #4 suggested asserting `GrassData->HasValidData()`, but `FLandscapeComponentGrassData` carries no `LANDSCAPE_API` on 5.3-5.8 (`LandscapeComponent.h:207`), so its out-of-line members do not link from a plugin module; `NumElements >= 0` is the same test written against a public data member (documented at `LandscapeComponent.h:230-233`) and is what both the fix and the tests use. No test id was added or renamed; `Content/Python/check_test_ids.py` re-run: `SCANNED 4793 id(s), 4793 unique ... CLEAN`. **Docs.** `Docs/wiki-src/landscape.md` gains a warning block on the grass section and rewrites the `landscape.flush_grass` page to state which of the two behaviours it has, carrying #3's and #4's measured numbers (922,280 -> 56,080 instances; 136 -> 0 components) as the reason not to reach for the console command. `Docs/rpc-design.md` §5b gains one bullet: an invalidation and a destruction empty the same cache, so a test that asserts the cache is empty proves neither — assert what SURVIVED; read the flag the engine passes at the equivalent moment, not the one a debug console command passes; a count of what went away is not coverage (`subObjectsRefreshed: 136` was a count of destroyed clusters published as refresh coverage); and a refresh hung off a generic seam inherits that seam's blast radius. **Not compiled, not run — orchestrator builds.** **Still not verified live, and that is the review to do:** nothing here re-measures the `0.4444 -> 0.4287` luminance claim, and nothing re-runs #4's capture to confirm the carpet is still there at the pixel level. A headless host cannot bake grass maps and place a camera. The tests prove the maps are not discarded and the instance cache is invalidated; a live pass at #4's pinned pose and pinned `ev100 -0.5` is what closes it. **State left by #4 is untouched by this change:** `/Game/Maps/PW_VegetationTest` in that editor has no grass until it restarts, and `grass.Enable` remains pinned at `ECVF_SetByConsole` for that session."
