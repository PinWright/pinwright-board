---
id: B-grass-varieties-edit-does-not-reach-renderer
title: "Editing a ULandscapeGrassType's GrassVarieties and saving reaches the renderer not at all (meanLuminance 0.4444 -> 0.4439 at a fixed pose), and the two flush paths a caller reaches for do not exist — grass.FlushCacheAll is EXEC_FAILED and unreal.Landscape.flush_grass_components is an AttributeError; only a raw system.console_command grass.FlushCache works"
status: IN-REVIEW
severity: High
category: bug
tags: [landscape, grass, vegetation, grass-type, silent-noop, derived-state, consumer-refresh, flush, misleading-success, material-consumers-precedent]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
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

**Workaround:** after any grass-type edit, `system.console_command {command: "grass.FlushCache"}`.
Measured effective: 0.4439 -> 0.4287.

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
