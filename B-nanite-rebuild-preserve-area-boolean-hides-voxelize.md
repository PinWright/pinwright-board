---
id: B-nanite-rebuild-preserve-area-boolean-hides-voxelize
title: "asset.nanite_rebuild_mesh collapses UE 5.8's three-valued ENaniteShapePreservation into one boolean whose default writes PreserveArea — the enumerator the engine labels '(Legacy foliage technique)' — leaving Voxelize ('for foliage that thins out otherwise') unreachable, silently overwriting the engine's None default plus PositionPrecision and GenerateFallback on every call, and echoing the request instead of a readback"
status: OPEN
severity: Medium
category: bug
tags: [nanite, asset, nanite_rebuild_mesh, static-mesh, foliage, vegetation, shape-preservation, voxelize, lumen, ray-tracing-proxy, unrequested-write, echo-not-readback, ue58]
---

# A boolean where the engine has three states, defaulted to the one marked legacy

UE 5.8 (`Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:3266-3274`):

```cpp
enum class ENaniteShapePreservation : uint8
{
    /** Do not attempt to preserve the object's shape in the distance. */
    None,
    /** Try to maintain the same surface area at all distances (Legacy foliage technique). */
    PreserveArea,
    /** Simplify triangles to voxels in the distance to preserve the perceived volume of the
        object. Useful for foliage that thins out otherwise. */
    Voxelize
};
```

`FMeshNaniteSettings::ShapePreservation` defaults to `None` (`:3315`).

`asset.nanite_rebuild_mesh` (`Handlers/Asset/AssetWorkflowHandler.cpp:1061`) exposes one boolean:

```
RPC_PARAM_OPT("preserveArea", "boolean",
    "When true (default) Nanite preserves total surface area during cluster simplification.")
```

`:1065`, defaulted true at `:1090`, and on UE >= 5.7 mapped at `:1106-1113`:

```cpp
if (bPreserveArea) { Settings.ShapePreservation = ENaniteShapePreservation::PreserveArea; }
else               { Settings.ShapePreservation = ENaniteShapePreservation::None; }
```

Three consequences, in increasing order of how hard they are to notice:

**1. `Voxelize` is unreachable.** A two-state input cannot select a third value, and the one the
engine documents *"for foliage that thins out otherwise"* — i.e. the case this ticket's research
pass was actually looking at — is the one that got dropped. The param doc describes the mechanism
accurately and never says the option it selects is the legacy one.

**2. Every call rewrites settings the caller did not mention.** `preserveArea` defaults to **true**
while the engine's own default is `None`, so a call made only to toggle Nanite or set a triangle
percentage — `{meshPath, trianglePercent: 50}` — also flips `ShapePreservation` from `None` to
`PreserveArea`. Two more are hardcoded in the same block: `Settings.PositionPrecision = 8` (`:1104`;
the engine default is `MIN_int32`, meaning *auto*, `EngineTypes.h:3317-3319`) and
`Settings.GenerateFallback` forced to `Enabled` or `PlatformDefault` from `fallbackPercent`
(`:1116-1123`). None of the three is a declared parameter of the thing the caller asked for.

**3. The response echoes the request, not the mesh.** `:1137-1143` returns `naniteEnabled`,
`preserveArea`, `trianglePercent`, `fallbackPercent` — every one of them the value that went in,
none read back off `StaticMesh->GetNaniteSettings()` after the write. `PositionPrecision` and
`GenerateFallback`, the two the caller never supplied, appear nowhere at all. This is the board's
recurring shape: the call succeeds, every reported number is correct as a statement about the
request, and the settings that decide the result are unreported.

## The foliage-specific Lumen fix that silently outranks all of it

`FMeshRayTracingProxySettings::FoliageOverOcclusionBias` — *"A bias to reduce foliage over occlusion
in Lumen GI. 0: no adjustment, 1: full strength"* — is `BlueprintReadWrite`, defaults to `0.0f`, and
lives at `EngineTypes.h:3554-3556`. **No PinWright verb writes it or reports it**: a grep for
`FoliageOverOcclusionBias` and `RayTracingProxySettings` across `Source/PinWright`,
`Source/PinWrightPCG` and `Source/PinWrightGeometry` returns zero hits.

It is not merely uncovered. It **takes priority over `ShapePreservation` entirely** —
`Engine/Source/Developer/NaniteBuilder/Private/Cluster.cpp:1054`:

```cpp
if ( RayTracingFallbackBuildSettings && RayTracingFallbackBuildSettings->FoliageOverOcclusionBias > 0.0f )
{
    ... ShrinkVoxelTriangles / ShrinkTriGroupWithMostSurfaceAreaLoss ...
}
else if( DAG.Settings.ShapePreservation == ENaniteShapePreservation::PreserveArea )   // :1065
{
    Simplifier.PreserveSurfaceArea();                                                  // :1067
}
```

So on any mesh where an artist set a non-zero bias, the `PreserveArea` this verb writes is **never
applied** — the `else if` is not reached. The verb reports `preserveArea: true`, the setting is
stored on the asset, and the Nanite build ignores it. A caller has no way to learn this from any
PinWright response, because nothing reports the bias.

## Fix

1. **Take an enum, not a boolean.** `shapePreservation` accepting `none` / `preserveArea` /
   `voxelize`, resolved by name with an error listing the valid values (the plugin's convention for
   an undiscoverable enum is already a live ticket, `E-pcg-set-self-pruning-enum-undiscoverable`).
   Keep `preserveArea` as a deprecated alias mapping to `preserveArea`/`none` so existing callers
   keep working, and **stop defaulting it**: when neither is supplied, leave
   `Settings.ShapePreservation` at whatever the asset already has. A rebuild verb should not silently
   restyle a mesh.
2. **Stop the other two unrequested writes.** Make `positionPrecision` a parameter defaulting to
   "leave unchanged" rather than a hardcoded `8`, and only touch `GenerateFallback` when
   `fallbackPercent` was actually supplied.
3. **Expose `foliageOverOcclusionBias`** as an optional parameter on the ray-tracing proxy settings,
   and — more important than the write — **report its effective value in the response**, with a
   warning when it is non-zero and `shapePreservation` was also requested, naming
   `Cluster.cpp:1054` as the reason the latter will be ignored. This is the one field that turns the
   response from an echo into a measurement.
4. **Read back after writing.** Publish the resolved `ShapePreservation`, `PositionPrecision`,
   `KeepPercentTriangles`, `FallbackPercentTriangles`, `GenerateFallback` and
   `FoliageOverOcclusionBias` from `GetNaniteSettings()` after `SetNaniteSettings`, not from the
   parsed payload.
5. **Document which option is which.** `Voxelize` is the modern foliage answer and `PreserveArea` is
   labelled legacy **in engine source**; a caller choosing between them from the wiki page currently
   has no way to know that.

## Correction to two existing tickets: these are two different verbs, not one alias

`B-render-nanite-rebuild-mesh-no-completion-signal` (DONE) and
`E-render-nanite-rebuild-async-poll-undocumented` (OPEN, Low) both state that
`render.nanite_rebuild_mesh` is *"also exposed as `asset.nanite_rebuild_mesh`"*. **They are separate
handlers with different behaviour**, and a fixer or doc author acting on that claim will write
something false:

| | `render.nanite_rebuild_mesh` | `asset.nanite_rebuild_mesh` |
|---|---|---|
| site | `Handlers/Render/RenderHandler.cpp:1860` | `Handlers/Asset/AssetWorkflowHandler.cpp:1061` |
| params | `assetPath` only | `meshPath`, `enableNanite`, `preserveArea`, `trianglePercent`, `fallbackPercent` |
| writes | `Settings.bEnabled = true` and nothing else (`:1891-1893`) | `bEnabled`, `ShapePreservation`, `PositionPrecision`, `KeepPercentTriangles`, `FallbackPercentTriangles`, `GenerateFallback` |
| async | `Ctx.StartJob` + `FJobBindArgs` + `AsyncTask`, returns a ticket | fully **synchronous**, `Ctx.SendSuccess` at `:1144`, no job |

The async ticket / `system.job_status` poll pattern those two tickets describe applies to the
`render` verb only. Recorded here rather than appended to them, since neither is this ticket's
subject; a reviewer who wants the correction on those files can lift this paragraph.

## Distinct from

- **`E-static-mesh-describe-doc-promises-nanite`** (OPEN, Low) — the readback side of the same gap:
  `static_mesh.describe`'s doc advertises Nanite state and the response carries none. Closest
  neighbour, and complementary: that ticket asks for Nanite state to be *readable*, this asks for it
  to be *writable correctly and reported after the write*. Landing both gives a caller a full
  round trip.
- **`B-render-nanite-rebuild-mesh-no-completion-signal`** (DONE) and
  **`E-render-nanite-rebuild-async-poll-undocumented`** (OPEN, Low) — async plumbing and its docs for
  the `render` verb. Neither touches settings, and see the correction above.
- **`E-geometry-convert-static-mesh-no-asset-echo`**, **`E-rendering-project-settings`** — adjacent
  Nanite mentions, unrelated axes.

## Not RPC-verified

Source-read only; the editor was not running and no `asset.nanite_rebuild_mesh` call was made, no
mesh was rebuilt, and no render was compared. Every mechanism claim is from engine and plugin source
at the cited lines. Two things an editor test would settle: whether `Voxelize` visibly outperforms
`PreserveArea` on the thin-foliage case the engine comment describes (which decides how loudly the
docs should steer), and whether a non-zero `FoliageOverOcclusionBias` on a real asset does suppress
the `PreserveArea` path end-to-end as `Cluster.cpp:1054` reads — the code is unambiguous but the
build path from `FMeshNaniteSettings` to `DAG.Settings` was not traced.

severity rationale: impact=Medium — soft blocker with a real if undocumented recovery: `property.set` accepts dotted nested paths on arbitrary UObjects (`Handlers/Utility/UtilityPropertyHandler.cpp:996-999`), so a caller who knows can write `NaniteSettings.ShapePreservation` directly and reach `Voxelize`; the case for High is genuine — the verb performs three writes the caller never requested (`ShapePreservation` flipped off the engine's `None` default, `PositionPrecision` forced to 8, `GenerateFallback` forced) and echoes the request rather than a readback, which is the rubric's "silent hardcoded data on a normal path" — and it is declined because the consequence is distant-LOD appearance rather than wrong data, and because every one of those writes is observable afterwards through `asset.dump` and reversible through `property.set` × reach=normal — `asset.nanite_rebuild_mesh` is standard mesh-prep rather than a rare edge path, so no modifier applies -> Medium

## History
- `#1-boolean-collapses-three-valued-enum` `OPEN` reporter — Source-read only, editor not running; no `asset.nanite_rebuild_mesh` call was made and no mesh was rebuilt. UE 5.8's `ENaniteShapePreservation` has three values (`EngineTypes.h:3266-3274`) with `None` as the `FMeshNaniteSettings` default (`:3315`); `PreserveArea` is labelled *"(Legacy foliage technique)"* in the engine's own comment (`:3270-3271`) and `Voxelize` is documented *"Useful for foliage that thins out otherwise"* (`:3272-3273`). `asset.nanite_rebuild_mesh` (`AssetWorkflowHandler.cpp:1061`) exposes one boolean `preserveArea` (`:1065`) defaulted true (`:1090`) that maps to `PreserveArea` / `None` (`:1106-1113`), so `Voxelize` is unreachable and every call rewrites `ShapePreservation` away from the engine default whether or not the caller mentioned it — alongside two more unrequested writes, `PositionPrecision = 8` (`:1104`, engine default `MIN_int32` = auto, `:3317-3319`) and a forced `GenerateFallback` (`:1116-1123`). The response (`:1137-1143`) echoes the four parsed inputs and reads nothing back; the two hardcoded settings appear nowhere. Separately, `FMeshRayTracingProxySettings::FoliageOverOcclusionBias` (`EngineTypes.h:3554-3556`, `BlueprintReadWrite`, default 0.0f) is written and reported by no verb — zero grep hits across all three plugin modules — and it TAKES PRIORITY over `ShapePreservation`: `Developer/NaniteBuilder/Private/Cluster.cpp:1054` tests the bias first and only falls through to the `PreserveArea` branch at `:1065-1067` when it is zero, so on a mesh with a non-zero bias the setting this verb writes is stored, echoed, and never applied. Destination decision: filed as a new ticket rather than an encounter, because the board has zero hits for `ENaniteShapePreservation`, `PreserveArea`, `Voxelize`, `FMeshRayTracingProxySettings` or `FoliageOverOcclusionBias`, and the three existing `nanite_rebuild` tickets are async-plumbing/docs. Filed under `B-` rather than the proposed `E-` because the substance is unrequested writes plus echo-not-readback on a mutating verb, not discoverability. Contradiction found on the board and recorded in the body rather than by editing those files: `B-render-nanite-rebuild-mesh-no-completion-signal` (DONE) and `E-render-nanite-rebuild-async-poll-undocumented` (OPEN) both say `render.nanite_rebuild_mesh` is "also exposed as `asset.nanite_rebuild_mesh`" — they are two different handlers with different parameters and different behaviour, and only the `render` one is async (`RenderHandler.cpp:1860`, `:1891-1893`, `Ctx.StartJob`; the asset one is synchronous, `Ctx.SendSuccess` at `AssetWorkflowHandler.cpp:1144`). Cross-linked to `E-static-mesh-describe-doc-promises-nanite` (OPEN, Low) as the readback half of the same gap.
