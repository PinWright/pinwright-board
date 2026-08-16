---
id: B-compile-material-landscape-consumers-stale
title: "compile_material reports compileSucceeded while a landscape master graph edit reaches nothing — no FMaterialUpdateContext and no combination-MIC invalidation"
status: DONE
severity: High
category: bug
tags: [material, material-authoring, landscape, compile, mic, shader-map, silent-noop, misleading-success, derived-state]
---

# `material.authoring.compile_material` compiles the master and refreshes no consumer

`material.authoring.compile_material` returned `compileSucceeded: true` for a graph edit to a
landscape master material that was **not picked up by the landscape at all**.

Proven with a destructive control rather than an opinion: forcing the lane gate to 0
everywhere — which must erase the entire painted road network — moved a fixed frame
**1.81%**, the noise floor. After an `assign a different material, assign the intended one
back` round-trip on `landscape_material`, the identical test moved **17.19%**.

Consequence: any master-material graph edit that is not round-tripped **measures its own work
as a no-op** while every verb in the chain reports success. An agent tuning a value concludes
it has no effect and keeps raising it.

## Root cause (source-verified against `C:\UE_5.8`)

Two independent gaps, same direction.

**A. No `FMaterialUpdateContext` around the master's change.**
`UMaterial::PostEditChangePropertyInternal` (`Material.cpp:5350`) regenerates `StateId` and
recompiles the master (`:5412-5433`) and recreates component render states (`:5431`) — but it
creates **no** `FMaterialUpdateContext`. `FMaterialUpdateContext::~FMaterialUpdateContext`
(`MaterialShared.cpp:5049`) is the only code that recaches dependent `UMaterialInstance`
static permutations: it iterates loaded instances whose base material changed (`:5099-5116`)
and calls `InitStaticPermutation` (`:5139`). Without it, every material instance keeps the
shader map it built against the previous master.

The engine's own recompile does exactly what the handler did not:
`UMaterialEditingLibrary::RecompileMaterialInternal` (`MaterialEditingLibrary.cpp:988-998`)
opens an `FMaterialUpdateContext`, `AddMaterial`, then `PreEditChange(nullptr)` +
`PostEditChange()`. So does the material editor's apply path (`MaterialEditor.cpp:2726-2732`).

**B. Landscape keeps a private cache no update context can see.**
`ALandscapeProxy::MaterialInstanceConstantMap` holds the per-weightmap-allocation combination
materials. `ULandscapeComponent::GetCombinationMaterial` reuses the cached entry whenever the
layer key and parent match (`LandscapeEdit.cpp:617-618`), so a graph edit keeps the
combination MIC and every component MIC parented to it. Only
`ALandscapeProxy::UpdateAllComponentMaterialInstances(bInInvalidateCombinationMaterials=true)`
resets it (`LandscapeEdit.cpp:845-850`).

## Why `11fe111a` does not cover this

`11fe111a` (`B-landscape-set-material-stale-mics`) taught `landscape.set_material` to build a
real `FPropertyChangedEvent` naming `LandscapeMaterial` and call the virtual
`PostEditChangeProperty`, so the engine's material branch runs and calls
`UpdateAllComponentMaterialInstances`. **The trigger is an assignment to that property.** A
graph edit to the master performs no assignment, so the mechanism never fires. Its *effect* is
reusable; its *trigger* is not. Our memory and handoff both recorded the round-trip as
obsolete after that commit — true for parameter/assignment changes, false for master edits.

## Fix

`compile_material` now closes both gaps and reports coverage as a measurement.

- `Source/PinWright/Private/Handlers/Material/MaterialLandscapeConsumers.h` (new):
  `PinWright::MaterialConsumers::RefreshLandscapeConsumers` walks
  `TActorIterator<ALandscapeProxy>` over the editor world, decides consumption with
  `ALandscapeProxy::RetrieveAllLandscapeMaterials` (`Landscape.cpp:5385`, `LANDSCAPE_API`,
  covers LandscapeMaterial + hole material + component overrides) plus
  `UMaterialInterface::GetMaterial()` so MIC chains resolve, and calls
  `UpdateAllComponentMaterialInstances(true)`.
- The refresh is **measured**: component `MaterialInstances` pointer sets are snapshotted
  before and after. `UpdateMaterialInstances_Internal` allocates a brand-new
  `ULandscapeMaterialInstanceConstant` per component per rebuild (`LandscapeEdit.cpp:715`), so
  an unchanged pointer set proves the rebuild did not run.
- `Source/PinWright/Private/Utils/DerivedStateReport.h` (new): shared `consumerRefresh` /
  `derivedWrite` wire vocabulary. `FConsumerRefreshReport` default-constructs to "nothing
  measured, nothing refreshed", so a path that forgets to measure reports failure.
- `MaterialAuthoringHandler.cpp`: `PreEditChange`/`PostEditChange` are scoped inside an
  `FMaterialUpdateContext`; the landscape refresh runs **before**
  `MaterialCompileErrorCollector::WaitAndCollect`, so the single `FinishAllCompilation()` also
  drains the MIC permutation jobs the rebuild queued. Response gains `consumerRefresh
  {measured, consumersFound, consumersRefreshed, consumersWithNothingToRefresh,
  subObjectsRefreshed, refreshed[], notRefreshed[], complete, remedy}` plus a `warnings[]`
  entry when coverage is incomplete or unmeasured.

Not done, deliberately: no dedicated `landscape.refresh_material` verb — `compile_material`
and `set_material` now both reach the rebuild.

## Acceptance

The round-trip becomes unnecessary for a **graph** edit: the lane-gate-to-0 control should
move the frame ~17% from a single `compile_material`, with
`consumerRefresh.complete: true` and the terrain named in `consumerRefresh.refreshed`.

## Related

- `B-landscape-set-material-stale-mics` — the assignment-path half, fixed in `11fe111a`. This
  ticket is the graph-edit path its mechanism cannot reach.
- `B-compile-material-false-shader-success` — same verb, different defect (missing shader
  error reporting). Untouched here.

## History
- `#1-initial-repro` `OPEN` reporter — `material.authoring.compile_material` returns `compileSucceeded:true` and a graph edit to `M_DotaTerrain` is not picked up by the landscape at all. Destructive control: forcing the lane gate to 0 everywhere (must erase the entire painted road network) moved the frame 1.81%, the noise floor; after an assign-away-and-back round-trip on `landscape_material` the identical test moved 17.19%. Consequence: any master-material graph edit that is not round-tripped measures its own work as a no-op while every verb reports success. Note that `11fe111a` was recorded as making the round-trip obsolete — that covers material-instance/assignment changes only.
- `#2-fix` `IN-REVIEW` developer — Root-caused to two gaps and fixed both. (A) `UMaterial::PostEditChangePropertyInternal` (`Material.cpp:5350`) recompiles the master but creates no `FMaterialUpdateContext`, and that destructor (`MaterialShared.cpp:5049,:5099-5156`) is the only code that recaches dependent material instances' static permutations; `compile_material` now scopes `PreEditChange`/`PostEditChange` inside one, matching `UMaterialEditingLibrary::RecompileMaterialInternal` (`MaterialEditingLibrary.cpp:988-998`). (B) `ALandscapeProxy::MaterialInstanceConstantMap` caches combination materials that no update context can see (`LandscapeEdit.cpp:617-618`); new `Handlers/Material/MaterialLandscapeConsumers.h` enumerates landscape consumers via `RetrieveAllLandscapeMaterials` (`Landscape.cpp:5385`) and calls `UpdateAllComponentMaterialInstances(true)` (`LandscapeEdit.cpp:845-850`). **Correction to the assumption in `#1`:** `11fe111a`'s mechanism is triggered by an assignment to `LandscapeMaterial`; a graph edit performs none, so it could never fire here — the effect transfers, the trigger does not. Coverage is measured by component-MIC object identity (`LandscapeEdit.cpp:715` allocates a fresh MIC per component per rebuild), not by counting rebuild calls, and published as `consumerRefresh` via the new shared `Utils/DerivedStateReport.h`; an incomplete or unmeasured refresh adds a `warnings[]` entry so the verb never reports a plain success it did not earn. Test: `Private/Tests/Material/TestCompileMaterialRefreshesLandscapeConsumers.cpp` seeds a sentinel into `MaterialInstanceConstantMap`, drives the production handler on the landscape's material, and asserts the sentinel is gone plus `consumersRefreshed >= 1`; a second case compiles a material no landscape uses and asserts `consumersFound == 0` with no warnings, so the verb cannot fabricate coverage. Docs: `wiki-src/material.authoring.md`, `wiki-src/landscape.md`, `wiki-src/level-building.terrain-and-water.md` now teach the single call and the `consumerRefresh.complete` check; `Docs/rpc-design.md` §5b records the transferable lesson. Not compiled or run — later integration phase.
- `#3-verified` `DONE` integrator — Compiled, suite-green and **proven live on the real terrain**. Build: clean module rebuild with intermediates moved aside, `-DisableAdaptiveUnity -NoHotReloadFromIDE`, `Result: Succeeded`, zero diagnostics in both the build log and UBT's `Log.txt`. Suite: **3760 tests performed, 3758 pass, 2 fail** — both the pre-existing `localization.Validation.*` — with `started == success + fail` and the ZenServer probe at 0; `PinWright.material.authoring.compile_material.RefreshesLandscapeConsumers` **passes** in 13 s. Runtime proof on `Dota2_Blockout` against `Landscape_0` / `DotaTerrain` with `M_DotaTerrain` (387 expressions), one fixed camera through `render.capture_open_level` (`-14500, 14500, 3000`, pitch -90, fov 70, 1280x720, Lit + game view, AA and shadows off for the measurement and restored after): a `Constant3Vector` of magenta wired straight into `Main.BaseColor`, then **`compile_material` alone with `save:false` and no `landscape.set_material`**. Noise floor 1.27% of pixels (MAD 0.795/255). **The decisive control the acceptance criteria did not ask for: a capture taken after the graph edit but BEFORE the compile measured 1.22% — at the noise floor.** The edit reached nothing on its own, and a forced viewport redraw did not deliver it; only `compile_material` did, at **95.26%** (93.16% even at threshold 64; mean luminance 0.4106 -> 0.4979). Restoring the original `BaseColor` connection and compiling again returned the frame to **1.20%** against the original, so the signal was the edit and not drift. Payload, identical on both calls: `consumerRefresh {measured:true, consumersFound:1, consumersRefreshed:1, consumersWithNothingToRefresh:0, subObjectsRefreshed:64, complete:true, refreshed:["DotaTerrain"], notRefreshed:["open asset editors..."]}`, no `warnings[]` key. Acceptance is met and exceeded — the criteria expected ~17% from the lane-gate control; a flatter control gave 95.26% against a 1.27% floor, ~75x. Open question #7 from the changeset (is `UpdateAllComponentMaterialInstances` safe inside the dispatcher's stack) is **clean for ordinary arrivals**: no hang, freeze or crash, the editor answered the next RPC immediately and reported `Responding: True`, the two calls took <=10.54 s and <=8.69 s wall (49 shaders, 4.97 s in-flight, dominant cost), and **zero** `Warning:`/`Error:` lines followed either call. The mid-tick arrival remains untested and is filed separately as `B-compile-material-not-tick-gated` (the verb is hazard family B per `Dispatch/SafePoint.cpp:24-36` and has no table entry). Nothing was saved: both calls used `save:false`, `M_DotaTerrain.uasset` on disk keeps its pre-session mtime, and the map was never dirtied. Shipped in `0fe35187`, pushed to origin/master as a fast-forward.
