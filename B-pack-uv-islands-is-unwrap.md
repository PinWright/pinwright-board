---
id: B-pack-uv-islands-is-unwrap
title: "geometry.pack_uv_islands does not pack islands — it re-runs the shared XAtlas unwrap, discarding existing UVs, and echoes a textureResolution no engine call ever receives"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, uv, pack_uv_islands, unwrap_uv, auto_uv, xatlas, dead-param, silent-wrong-data, LayoutMeshUVs, RepackMeshUVs]
---

# `geometry.pack_uv_islands` performs an unwrap, not a pack, and its `textureResolution` reaches no engine call

Two defects in one handler, both silent.

**1. The verb does not pack.** `geometry.pack_uv_islands`
(`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshInfoHandler.cpp:689-719`)
delegates to `GeometryUtils::ApplyXAtlasUnwrap`
(`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryUtils.cpp:152-197`),
the same routine `geometry.unwrap_uv` (`MeshInfoHandler.cpp:660-685`) and
`geometry.auto_uv` (`MeshOpsHandler.cpp:349-367`) call. All three run
`AutoGenerateXAtlasMeshUVs(Mesh, UVChannel, FGeometryScriptXAtlasOptions(), nullptr)`
(`GeometryUtils.cpp:175-176`) and differ only in their success message and, for
`pack_uv_islands`, one extra echoed field. The three verbs are behaviourally
identical.

That is not a naming quibble: an XAtlas auto-unwrap **recomputes the UV layer from
scratch**, so calling `pack_uv_islands` after seating UVs destroys them. The
documented pipeline in `F-geometry-uv-prep-pipeline-batch` is
unwrap → project → transform → pack; its final step silently discards the
`project_uv` and `transform_uvs` stages that preceded it and replaces them with a
fresh atlas. The caller reads `"UV islands packed"` and `success:true` on a mesh
whose carefully-authored UVs were just overwritten.

**2. `textureResolution` is a dead parameter presented as a live one.** It is
declared (`MeshInfoHandler.cpp:693`), read (`:698`), and written into the response
JSON (`:714-717`) — but never passed to any engine call. `ApplyXAtlasUnwrap` takes
no resolution argument, and `FGeometryScriptXAtlasOptions` is default-constructed at
the call site. A caller passing `textureResolution: 2048` gets back
`textureResolution: 2048` and has every reason to believe the atlas was sized for
it. Nothing was sized for anything. This is the "latent no-op param worth its own
ticket" flagged out of scope in `E-geometry-auto-uv-redundant-with-unwrap-uv`
(scope note, `#2`).

## Why the echo is the sharper half

A wrong operation can be caught by looking at the mesh. A parameter echoed back
unchanged is a **confirmation**, and the response is the only channel an agent has —
it cannot inspect the atlas. The reply is indistinguishable from one where the value
took effect, so nothing downstream can detect the discrepancy.

## What it should do

Pack the existing islands in `UVChannel` at the requested resolution, without
regenerating them. Two engine functions do this, and **no RPC verb reaches either** —
grep over `Source/` for
`RepackMeshUVs|LayoutMeshUVs|FGeometryScriptLayoutUVsOptions|FGeometryScriptRepackUVsOptions`
hits exactly one call site, the `.pwmodel` compiler: `FCompiler::ApplyUVOp`
(`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:716-779`) calls
`LayoutMeshUVs` on its `uv mode=layout` branch (`:761-767`) with
`FGeometryScriptLayoutUVsOptions::TextureResolution` wired through from the op's
`texture_resolution`, and ensures the layer first (`:725-729`). That branch is a
working reference implementation of exactly the fix below — copy it. (`RepackMeshUVs`
and `FGeometryScriptRepackUVsOptions` remain unused anywhere in `Source/`.) The two
candidates:

- `UGeometryScriptLibrary_MeshUVFunctions::LayoutMeshUVs(TargetMesh, UVSetIndex, FGeometryScriptLayoutUVsOptions, FGeometryScriptMeshSelection, Debug)`
  — `MeshUVFunctions.h:556-563`. The richer one, and the direct fit:
  `FGeometryScriptLayoutUVsOptions` (`:41-`) carries `TextureResolution` (default
  1024, matching this handler's default), plus `LayoutType`
  (`Transform | Stack | Repack | Normalize`, default `Repack`), `Scale`,
  `Translation`, `bPreserveScale`, `bPreserveRotation`.
- `UGeometryScriptLibrary_MeshUVFunctions::RepackMeshUVs(TargetMesh, UVSetIndex, FGeometryScriptRepackUVsOptions, Debug)`
  — `MeshUVFunctions.h:545-551`. Narrower: `FGeometryScriptRepackUVsOptions`
  (`:16-26`) has only `TargetImageWidth` and `bOptimizeIslandRotation`.

Engine paths read from
`C:/UE_5.8/Engine/Plugins/Runtime/GeometryScripting/Source/GeometryScriptingCore/Public/GeometryScript/MeshUVFunctions.h`.

**Fix:** point `pack_uv_islands` at `LayoutMeshUVs` with
`FGeometryScriptLayoutUVsOptions::TextureResolution = TextureResolution`, and pass a
real `UGeometryScriptDebug` rather than `nullptr` so a failure surfaces as an error
instead of a success echo. Leave `unwrap_uv` / `auto_uv` on `ApplyXAtlasUnwrap` —
they *are* unwraps and the shared routine is correct for them. Once the verbs
diverge, the three-way redundancy noted in
`E-geometry-auto-uv-redundant-with-unwrap-uv` collapses to the intended two-way
alias pair, and the ordering rationale in `F-geometry-uv-prep-pipeline-batch`
becomes true rather than aspirational.

`pack_uv_islands` must still ensure the target UV layer exists before laying it out
— packing a layer with no islands is the same silent-no-op class as
`B-geometry-uv-gen-silent-noop`, whose `EnsureMeshHasUVChannel` guard currently sits
inside `ApplyXAtlasUnwrap` (`GeometryUtils.cpp:168-173`) and would no longer be on
this verb's path.

## Not duplicates

- `E-geometry-auto-uv-redundant-with-unwrap-uv` — the `auto_uv`/`unwrap_uv`
  discovery tax, fixed by converging those two on a shared param and reciprocal
  summaries. It explicitly deferred the `textureResolution` no-op to a separate
  ticket and treated `pack_uv_islands` as "a distinct op, left as-is". This ticket
  is the claim that it is *not* a distinct op today, and that it should be.
- `B-geometry-uv-gen-silent-noop` — the whole UV-gen family creating zero elements
  when the layer is absent; fixed by `EnsureMeshHasUVChannel`. Orthogonal: that is
  about the layer not existing, this is about the wrong operation running on a layer
  that does.
- `F-geometry-uv-prep-pipeline-batch` — call-count friction across the 4-step
  pipeline. Names the same verbs; asks for a composite, not for any step to behave
  differently.

## Severity

impact = silent wrong data on a normal path (an echoed parameter that reached no
engine call, plus a destructive re-unwrap reported as a pack) × reach = normal (UV
prep is an ordinary authoring path; `F-geometry-uv-prep-pipeline-batch` records two
separate tasks reaching this verb) → **High**.

## Provenance

Found by source inspection while planning the `.pwmodel` model-format work, which
audits the `geometry.*` verb set. Not reproduced live — the citations above are read
from current plugin source and current engine headers, and the defect is visible in
the control flow without a run. A live repro would be: `unwrap_uv`, then `project_uv`
cylindrical, then `pack_uv_islands` with `textureResolution: 2048`, then read the UVs
back and observe the cylindrical projection is gone.

## History
- `#1-source-audit-two-defects` `OPEN` reporter — Filed from source inspection (no live repro; see Provenance). `geometry.pack_uv_islands` (`MeshInfoHandler.cpp:689-719`) delegates to `GeometryUtils::ApplyXAtlasUnwrap` (`GeometryUtils.cpp:152-197`), the same XAtlas auto-unwrap routine backing `unwrap_uv` and `auto_uv`, so all three verbs are behaviourally identical and the "pack" recomputes the UV layer instead of packing it — silently discarding any prior `project_uv`/`transform_uvs` result. Separately, `textureResolution` is read at `:698` and echoed at `:714-717` but reaches no engine call, so the response confirms a value the engine never saw. Fix candidates `LayoutMeshUVs` (`MeshUVFunctions.h:556-563`, whose `FGeometryScriptLayoutUVsOptions` carries `TextureResolution`) and `RepackMeshUVs` (`:545-551`) are unused anywhere in the plugin. Not fixed here — filed only, per the plan chunk that produced it. Distinct from `E-geometry-auto-uv-redundant-with-unwrap-uv` (which deferred this exact `textureResolution` finding), `B-geometry-uv-gen-silent-noop` (missing UV layer, not wrong op), and `F-geometry-uv-prep-pipeline-batch` (call-count friction).
- `#2-reanchor-cites-and-correct-grep` `OPEN` reporter — Reworded, still OPEN, defect not fixed. (a) **Citations re-anchored.** Both cited files were rewritten by later waves of the same plan, so every `#1` line number has drifted; per the `E-geometry-auto-uv-redundant-with-unwrap-uv` precedent (`#2`) the as-filed numbers stay in the body and the current coordinates are recorded here. Measured against the post-refactor tree (`MeshInfoHandler.cpp` is now 597 lines, was 719): `pack_uv_islands` handler `MeshInfoHandler.cpp:689-719` → **`:567-597`** (banner comment `:564-566`); `textureResolution` declared `:693` → **`:571`**, read `:698` → **`:576`**, echoed `:714-717` → **`:590-594`** (the echo is now an `ExtraFields` lambda passed to `ApplyXAtlasUnwrap`); `unwrap_uv` `:660-685` → **`:538-562`**; `auto_uv` `MeshOpsHandler.cpp:349-367` → **`:281-312`**, and it no longer routes through `ApplyXAtlasUnwrap` at all — it calls `GeometryOps::UnwrapUVXAtlas` directly (`:298`). `GeometryUtils::ApplyXAtlasUnwrap` `GeometryUtils.cpp:152-197` → **`:153-181`** (primary) plus the thin overload **`:183-193`**; it is now only a `Ctx`-bound notify/respond shell. The two anchors cited inside it no longer exist there: the `AutoGenerateXAtlasMeshUVs(Mesh, UVChannel, FGeometryScriptXAtlasOptions(), nullptr)` call (`GeometryUtils.cpp:175-176`) moved into `GeometryOps::UnwrapUVXAtlas`, **`GeometryOps_Modeling.cpp:1069-1070`** (function `:1046-1074`), and the `EnsureMeshHasUVChannel` guard (`GeometryUtils.cpp:168-173`) moved with it, **`GeometryOps_Modeling.cpp:1063-1067`**. The closing argument is unaffected and if anything sharper: the guard now sits on the XAtlas op itself, not on the shared shell, so a `pack_uv_islands` retargeted at `LayoutMeshUVs` leaves that path entirely and must call `GeometryUtils::EnsureMeshHasUVChannel` (`GeometryUtils.cpp:269`) itself. Engine-header citations re-verified **unchanged**: `MeshUVFunctions.h` `RepackMeshUVs` `:547`, `LayoutMeshUVs` `:558`, `FGeometryScriptRepackUVsOptions` `:17` (`TargetImageWidth` `:22`), `FGeometryScriptLayoutUVsOptions` `:42` (`TextureResolution` default 1024, `:52`). (b) **Grep assertion corrected in the body.** `#1` and the "What it should do" section claimed the four symbols return zero hits over `Source/` and that `LayoutMeshUVs`/`RepackMeshUVs` are "unused anywhere in the plugin" — true when filed, false now, and a fixer re-running that grep would find the ticket contradicted at its load-bearing claim. Corrected to: no *RPC verb* reaches either function; the sole call site is the `.pwmodel` compiler's `uv mode=layout` path, `FCompiler::ApplyUVOp` (`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:716-779`), which calls `LayoutMeshUVs` at `:765-766` with `FGeometryScriptLayoutUVsOptions::TextureResolution` wired from the op's `texture_resolution` (`:763-764`) and ensures the UV layer before dispatching any mode (`:725-729`). That is a working reference implementation of precisely the fix this ticket prescribes — including the layer-ensure placement the closing argument asks for — so the fix is a port of that branch, not new design. `PwModelParser.cpp:391` already notes in-source that `LayoutMeshUVs` is reached by no RPC verb. `RepackMeshUVs` and `FGeometryScriptRepackUVsOptions` still return zero hits across `Source/`. No prior history entry altered; no code changed.
- `#3-review-a-current` `OPEN` reporter — Additional evidence: **Adversarial review A — confirmed current; citation drift noted.** Actuality: CONFIRMED CURRENT. Framing: title and High severity remain accurate for silent wrong UV data, but the claimed 597-line post-refactor/`GeometryOps_Modeling`/`PwModelCompiler` tree is absent on current `b16f0f2b`; the live handler is still `MeshInfoHandler.cpp:689-719`. Proposed fix: INCOMPLETE, `LayoutMeshUVs`/`RepackMeshUVs` plus the existing UV-layer guard is the right operation-level repair, but passing a real `UGeometryScriptDebug` alone cannot surface failure while the helper still unconditionally sends success. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshInfoHandler.cpp:689-717` and `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\GeometryUtils.cpp:161-184` show the pack path and unconditional XAtlas success; `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshUVFunctions.cpp:1181-1189` then `:1189-1227` show XAtlas recomputes and clears the UV overlay, while `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Public\GeometryScript\MeshUVFunctions.h:42-52,547-563` and `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshUVFunctions.cpp:933-1052` provide existing-UV pack/layout APIs with resolution. Current coverage is only the shared unwrap path (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Geometry\TestGeometryUvGenerationOnUVLessMesh.cpp:253-269`) plus pack missing-param registration (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\World\TestGeometryHandlers.cpp:572-583`); no pack-preservation or resolution test exists. Runtime: NOT VERIFIED. Recommendation: KEEP; retarget pack to an existing-UV layout/repack call, retain `EnsureMeshHasUVChannel`, inspect debug/postconditions before success, and add a pack-specific regression that seeds authored UVs and verifies they are packed rather than regenerated.
- `#4-adversarial-reframe` `OPEN` reporter — Additional evidence: **Adversarial review B — agrees with A on current actuality, absent pack-specific coverage, and the need to inspect operation errors; disagrees that the title is fully precise and that retaining only the layer guard is sufficient.** Actuality: CONFIRMED CURRENT. Framing: High impact remains correct, but XAtlas does create a packed atlas (`XAtlasPackOptions`), so “does not pack islands” overstates the defect; reframe it as re-unwrap/replacement instead of repacking existing UVs, plus a dead resolution echo. Proposed fix: INCOMPLETE, `RepackMeshUVs` is the narrower existing-island operation (`TargetImageWidth`), with `LayoutMeshUVs` as an alternative; `EnsureMeshHasUVChannel` alone creates an element-less layer and can still yield an empty-pack success, so reject missing/empty target UV islands and update the shared UV-generation test assumptions when pack leaves `ApplyXAtlasUnwrap`. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshInfoHandler.cpp:689-717`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\GeometryUtils.cpp:161-184`; `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshUVFunctions.cpp:933-967,1127-1190`; `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryAlgorithms\Public\XAtlasWrapper.h:29-40`; current tests `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Geometry\TestGeometryUvGenerationOnUVLessMesh.cpp:253-269` and `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\World\TestGeometryHandlers.cpp:572-583` cover no pack-preservation/resolution contract. Runtime: NOT VERIFIED. Recommendation: REFRAME; keep OPEN, implement an operation-specific repack path with real error/postcondition handling and authored-UV/resolution/empty-input regression coverage.

- `#5-retargeted-onto-layoutuv` `IN-REVIEW` developer — Chose **implement**, not re-document. Re-documenting would have been honest but was not the cheaper option here: `GeometryOps::LayoutUV` already exists in Repack mode, wired field-for-field to `FGeometryScriptLayoutUVsOptions` and used by the .pwmodel compiler, so a real pack was a re-route rather than new design — exactly the port review A and the ticket body both prescribed. `geometry.pack_uv_islands` (MeshInfoHandler.cpp) now calls `GeometryOps::LayoutUV` with `LayoutType=Repack` and `TextureResolution` carried into the options, so the echoed value is one the engine saw; `unwrap_uv`/`auto_uv` keep the XAtlas path, which is correct for them. Review B's objection is handled: `EnsureMeshHasUVChannel` alone would create an element-less layer and pack zero islands as a success, so the handler checks `GeometryUtils::MeshHasUVElementsInChannel` FIRST and refuses with INVALID_ARGUMENT naming unwrap_uv/project_uv. A non-positive `textureResolution` is refused rather than silently defaulted. Two stale in-source comments claiming pack shares the unwrap shell are corrected. Three failure-direction tests in Tests/Geometry/TestGeometryPackUVIslandsRepacks.cpp: `.PackRearrangesTheLayoutInsteadOfReplacingIt` separates repacked from re-unwrapped using a signal neither review had — a planar projection leaves faces parallel to the projection axis at zero UV area, a repack keeps them degenerate, an XAtlas unwrap drives the count to zero — and measures that degenerate count on the fixture as a precondition rather than assuming it; `.TextureResolutionReachesThePacker` packs the same fixture at 64 and 4096 and requires the results to differ, which an echoed-and-dropped value cannot do; `.EmptyChannelIsRefusedNotPackedAsNothing` covers the refusal. Docs/wiki-src/geometry.md: the three places calling pack an alias of the unwrap are corrected and a `### geometry.pack_uv_islands` section added. Commits d40ebe1d (code+tests) and f50c74e1 (docs). Both touched TUs compile-checked -SingleFile, [1/1] Compile, Result: Succeeded. Not yet run in a suite.
