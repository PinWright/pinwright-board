---
id: B-compile-mgir-landscape-consumers-stale
title: "material.compile_mgir compiles and SAVES a landscape master while the landscape ignores it — the door 0fe35187 did not close"
status: IN-REVIEW
severity: High
category: bug
tags: [material, mgir, landscape, compile, mic, shader-map, silent-noop, misleading-success, derived-state, incomplete-fix]
---

# `material.compile_mgir` writes a correct `.uasset` and reaches no landscape

`material.compile_mgir` returned `blocksCompiled: 1`, `expressionsCreated: 1` and a real
`assetPaths[]` for a graph compile against a landscape master material that the landscape
**did not pick up at all** — and, with the default `save: true`, wrote the correct asset to
disk while the terrain kept rendering the previous shader map.

This is the **same defect** as `B-compile-material-landscape-consumers-stale`, reached
through a different verb. That ticket was closed `DONE` on the strength of a live
destructive control; the control exercised `compile_material` only, and the fix was applied
only there. `material.compile_mgir` is the bulk text-IR path the wiki **actively recommends
over** repeated imperative calls, so the recommended authoring route was the one still broken.

## Measured evidence (live, real terrain)

`Dota2_Blockout` / `DotaTerrain` / `M_DotaTerrain` (389 expressions, **zero named
parameters** — confirmed by `get_material_info` returning `parameters: []`, so there is no
material-instance route to this master at all). One fixed camera, `render.capture_open_level`
at `(-14500, 14500, 3000)`, pitch -90, fov 70, 1280x720. Metric: share of pixels whose max
per-channel difference exceeds 24, via `Docs/scripts/analysis/frame_delta.py` in the host
project.

| Step | Frame delta |
|---|---|
| noise floor (two captures, same state) | **0.18%** |
| graph edit alone via `connect_nodes` (magenta into `BaseColor`) | **0.02%** |
| same edit + `material.authoring.compile_material` | **93.82%** |
| identical MGIR document, **pre-fix** `compile_mgir` | **0.79%** |
| identical MGIR document, **post-fix** `compile_mgir` | **93.70%** |

The edit provably landed in every case: `get_material_info` showed `nodeCount` 389 -> 390 and
`BaseColor` rewired to the new magenta node. So 0.79% is not a failed edit — it is a landed
edit that reached nothing.

### Exposure was controlled for, two independent ways

Auto-exposure is live in this project (`r.EyeAdaptationQuality` reads `2`), and a sibling
workstream nearly filed a false plugin bug after a real material change was hidden by
re-exposure. Both checks were run here, and **neither collapses the gap**:

1. **Gain-invariant re-analysis of the captures.** A pure exposure difference is a scalar
   gain, so it must vanish once the least-squares best-fit gain is divided out.
   `frame_delta.py` now fits and removes it. The no-op frames have best-fit gain **0.9996**
   and **0.9972** — auto-exposure barely moved between them, so there was no exposure change
   available to hide anything — and their residuals stay at 0.02% / 0.81%. The pushed frames
   need gain 1.685 and their residual *rises* from 93.82% to **99.44%**: no scalar gain can
   reconcile them, which is the signature of a content change rather than a re-exposure.
2. **Live re-run with exposure pinned** (`r.EyeAdaptationQuality 0`, read back as `0`,
   restored to `2` afterwards):

| Step, exposure pinned | Frame delta | after removing best-fit gain |
|---|---|---|
| unpushed edit (`connect_nodes` only) | **0.04%** | 0.05% |
| same edit + `compile_material` | **94.12%** | 99.92% |

The control is also chosen to be exposure-proof by construction: it wires **pure magenta**
into `BaseColor`, a hue change. A scalar exposure gain cannot map magenta back to terrain
colours, and the metric is per-channel, so the signal survives any gain. A luminance-shaped
control such as "force the lane gate to 0" does not have that property — which is why the
original ticket's 1.81% / 17.19% figures were never used or relied on here.

## Root cause

`FinalizeMaterial` (`Source/PinWright/Private/MGIR/MGIRCompiler.cpp:473`) did:

```cpp
Material->PreEditChange(nullptr);
Material->PostEditChange();
Material->ForceRecompileForRendering();
```

Strictly less than the **pre-fix** `compile_material` had done, and missing both halves:

- **No `FMaterialUpdateContext`**, so dependent `UMaterialInstance` static permutations are
  never recached (`MaterialShared.cpp:5099-5156` is the only code that does it). The
  *function* twin in the same file (`:522-528`) does build one — the material side was the
  oversight.
- **No landscape rebuild**, so `ALandscapeProxy::MaterialInstanceConstantMap` keeps the
  combination materials it built against the previous graph. In `Append` mode (the default)
  the compile first empties and rebuilds the entire expression collection
  (`ClearMaterialGraph`), which is precisely the layer-allocation change those cached MICs
  are keyed on (`LandscapeEdit.cpp:617-618`) — the worst case, not an edge of it.

It then saved the asset (`:499`), so every durable signal said success.

## Fix (shipped `f92a4d32`)

- `Handlers/Material/MaterialLandscapeConsumers.h` gains `NotifyMasterMaterialChanged`
  (update-context-scoped `PreEditChange`/`PostEditChange`, matching
  `UMaterialEditingLibrary::RecompileMaterialInternal`, `MaterialEditingLibrary.cpp:988-998`)
  and `ApplyMasterMaterialEdit` (that plus the measured landscape rebuild), so the pairing has
  **one name** and the next verb cannot ship half of it.
- `MGIRCompiler.cpp` `FinalizeMaterial` uses it; the landscape rebuild runs **after**
  `ForceRecompileForRendering` so components build against a current master.
- `FMGIRCompileResult` / `FMGIRCompiledBlock` carry `FConsumerRefreshReport`;
  `FConsumerRefreshReport::Accumulate` folds per-block coverage (counts sum, names
  `AddUnique`) for documents compiling several materials.
- `MGIRCompileHandler.cpp` publishes `consumerRefresh` plus a `warnings[]` entry when
  coverage is incomplete or unmeasured.
- `material.authoring.configure_layer_blend` — the verb whose only purpose is landscape layer
  blending — had the identical bare `PostEditChange()` and is fixed the same way.
- `compile_material` is refactored onto the shared helper so the verbs cannot drift.

## Deliberately NOT fixed — named, not hidden

The incremental graph mutators do **not** push, by design: a full component-MIC rebuild
allocates 64 fresh `UObject`s plus permutation jobs, so firing it per node insertion would
make batch authoring pathological. Unpushed: `connect_nodes`, the `add_*` family, all of
`material.graph.*` (`create_nodes`, `add_expression`, `add_texture_sample`, `add_node`,
`remove_node`, `break_connections`), `set_blend_mode` / `set_shading_model` /
`set_material_domain` / `set_two_sided`, `set_material_layer_stack`,
`set_texture_sample_texture`, `add_collection_parameter_node`, and generic `property.set` /
`property.reset` / `container.*` on a `UMaterial` (`UtilityPropertyHandler.cpp:1129`).

The line drawn and published: **verbs that COMPLETE a unit of work push and report
`consumerRefresh`; verbs one step inside a unit of work do not.**
`Docs/wiki-src/material.authoring.md` names both lists under `## Limitations and reliability
notes` (above the first `###`, so it renders on the namespace page) and carries the
0.02% / 93.82% control showing what an unpushed edit measures as.

## Verification

- Build: `EAContentExamples58Editor Win64 Development`, `Result: Succeeded`, zero warnings.
- Suite: **3761 tests performed, 3761 pass, 0 fail**, `started == success + fail`, ZenServer
  probe 0, exit code 0. Derived expectation 3760 (measured at `0fe35187`) + 1 new test = 3761,
  matched exactly. The two previously-failing `localization.Validation.*` tests now pass
  (another agent's concurrent work added the missing `Config/Localization/`).
- Live: the destructive control above, restored afterwards to 0.09% of baseline. The terrain
  master was never left modified — `M_DotaTerrain.uasset` is byte-identical to `HEAD`.

## Related

- `B-compile-material-landscape-consumers-stale` — same defect, `compile_material` door,
  fixed in `0fe35187`. Its `DONE` verdict was correct for the verb it tested and wrong as a
  verdict on the defect. Not reopened; this ticket is the door it missed.
- `B-compile-material-not-tick-gated` — the rebuild is now reachable from a second verb, so
  the untested mid-tick-arrival hazard has one more entry point.

## History
- `#1-sibling-found` `OPEN` reporter — Found while verifying `B-compile-material-landscape-consumers-stale`. `RefreshLandscapeConsumers` had exactly **one** call site across 99 material verbs (`MaterialAuthoringHandler.cpp:2716`). `material.compile_mgir` reaches the same master through `FinalizeMaterial` (`MGIRCompiler.cpp:473`) with a bare `PreEditChange(nullptr)` + `PostEditChange()` + `ForceRecompileForRendering()` and no consumer refresh, then saves. Reproduced live on `M_DotaTerrain`: identical MGIR document moved a fixed frame **0.79%** against a **0.18%** noise floor, with `nodeCount` 389 -> 390 and `BaseColor` rewired proving the edit landed. The bug the parent ticket was closed on was still reachable through the path the wiki recommends.
- `#2-exposure-challenged-and-cleared` `OPEN` reporter — The evidence was challenged on the grounds that a sibling workstream had just found a real material change measuring as "no effect" purely because the capture re-exposed. Checked rather than argued, two ways, and **the gap does not collapse**: (a) removing the least-squares best-fit scalar gain from the existing captures leaves the no-ops at 0.02% / 0.81% (their best-fit gains are 0.9996 / 0.9972, i.e. auto-exposure barely moved, so there was nothing for it to hide) while the pushed frames' residual *rises* to 99.44% because no gain can reconcile them; (b) re-running live with `r.EyeAdaptationQuality 0` (read back, then restored to `2`) gives **0.04% unpushed vs 94.12% pushed**. The control is exposure-proof by construction — pure magenta into `BaseColor` is a hue change and the metric is per-channel, so a scalar gain cannot mask it. Note for the record: the original parent-ticket figures (1.81% / 17.19%) come from a luminance-shaped "lane gate to 0" control that does *not* have that property; they were never used or relied on in this ticket, and remain unre-verified.
- `#3-fix` `IN-REVIEW` developer — Fixed as described above and shipped in `f92a4d32` (pushed to `origin/master` as a fast-forward). Post-fix the identical document moves the frame **93.70%** with `consumerRefresh {measured:true, consumersFound:1, consumersRefreshed:1, subObjectsRefreshed:64, complete:true, refreshed:["DotaTerrain"]}` and no `warnings[]`, from a single `compile_mgir` call with no round-trip and no `compile_material`. Regression test `PinWright.material.compile_mgir.RefreshesLandscapeConsumers` seeds a sentinel into `MaterialInstanceConstantMap`, drives the production handler, and asserts the sentinel is gone plus measured coverage — the same differential property as its `compile_material` sibling so the two cannot drift; it also asserts `blocksCompiled >= 1` so a refresh cannot be claimed for a compile that did nothing. Suite **3761/3761 green**. Scope is explicit rather than assumed: the incremental mutators are listed as deliberately unpushed with the performance reason, in the ticket and on the namespace wiki page. `Docs/rpc-design.md` §5b gains the transferable lesson — *fixing one entry point is not fixing the defect; enumerate every door into the same engine state before calling it closed* — which is the exact mistake that let this survive.
