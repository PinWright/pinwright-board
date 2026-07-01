---
id: B-asset-dump-mic-no-material-instance-json
title: "asset.dump emits no material_instance.json for UMaterialInstanceConstant assets"
status: DONE
severity: High
category: bug
tags: [asset-dump, material, material-instance, coverage-gap]
---

# asset.dump emits no material_instance.json for UMaterialInstanceConstant assets

`asset.dump` / `asset.dump_folder` dump `UMaterialInstanceConstant` (MIC)
assets through the generic aspect path only:

```text
meta.json
properties.json
```

There is no `material_instance.json` sidecar, even though MICs are the
dominant material kind in the project — ~6× the count of base
`UMaterial` assets. From the live `App/` dump:

- 366 MIC folders under `App/App/Drone/.../Materials/`
- 439 MIC folders under `App/Megascans/`
- 3 MIC folders under `App/App/MWLandscapeAutoMaterial/`
- More scattered through the corpus

→ **~811+ MIC asset folders ship with zero structured override data.**

`CLAUDE.local.md` lists `material_instance.json` as the canonical
per-type sidecar for MICs, so this is a real handler gap, not a doc
error. The sibling work for base materials (`B-asset-dump-material-mgir-aspect`,
DONE) added `mgir.txt` for `UMaterial` and `UMaterialFunction` via the
`REGISTER_DECOMPILE_IR` plumbing — but neither that registration nor an
explicit branch in `AssetDumpHandler` exists for `UMaterialInstanceConstant`.

## Repro

Dump `/App` via `asset.dump_folder` (or any single MIC via `asset.dump`),
then inspect any MIC folder. Confirmed live on:

```text
.editor-automation/asset-dumps/App/App/Drone/Geoscan_801/MI_CarbonFiber3/
    meta.json
    properties.json

.editor-automation/asset-dumps/App/Megascans/3D_Assets/Old_Concrete_Barrier_vksrdes/MI_Old_Concrete_Barrier_vksrdes_8K/
    meta.json
    properties.json

.editor-automation/asset-dumps/App/Megascans/3D_Assets/Sharp_Cliff_rcygn/Preview/MI_Sharp_Cliff_rcygn_1K/
    meta.json
    properties.json
```

All three have only the generic pair. No `material_instance.json`.

## Why properties.json alone is insufficient

The current `properties.json` for an MIC stores override data as
serialized export-text strings, not structured fields. Sample from
`MI_CarbonFiber3`:

```json
"BasePropertyOverrides": {
    "value": "(BlendMode=BLEND_Masked,ShadingModel=MSM_FromMaterialExpression,OpacityMaskClipValue=0.333300,DisplacementScaling=(Magnitude=4.000000,Center=0.500000),DisplacementFadeRange=(StartSizePixels=4.000000,EndSizePixels=1.000000))"
},
"EditorOnlyData": {
    "value": {
        "StaticParameters": "(MaterialLayers=())"
    }
}
```

Consumers must regex-scrape these strings to recover individual fields
(`BlendMode`, `OpacityMaskClipValue`, static switch values, layer stacks,
…). Per-parameter scalar/vector/texture override arrays (`ScalarParameterValues`,
`VectorParameterValues`, `TextureParameterValues`) similarly land as
generic struct-array dumps without the resolved parameter info / guid /
value triple in a stable shape.

## Expected

For every dumped `UMaterialInstanceConstant`, write a structured
`material_instance.json` sidecar containing at minimum:

- `parent` — asset path of the parent `UMaterialInterface`
- `parentChain` — flattened chain up to the base `UMaterial` (optional but
  useful: MIC → MIC → … → Material)
- `overrides`:
  - `scalar: { Name: Value }` from `ScalarParameterValues`
  - `vector: { Name: {R,G,B,A} }` from `VectorParameterValues`
  - `texture: { Name: AssetPath }` from `TextureParameterValues`
  - `staticSwitch: { Name: Bool }` from `StaticParameters.StaticSwitchParameters`
  - `staticComponentMask: { Name: {R,G,B,A: Bool} }` from
    `StaticParameters.StaticComponentMaskParameters`
- `basePropertyOverrides` — structured `FMaterialInstanceBasePropertyOverrides`:
  `bOverride_*` flags plus per-field values
  (`BlendMode`, `ShadingModel`, `OpacityMaskClipValue`,
  `TwoSided`, `DitheredLODTransition`, `OutputTranslucentVelocity`,
  `CastDynamicShadowAsMasked`, `DisplacementScaling.{Magnitude,Center}`,
  `DisplacementFadeRange.{StartSizePixels,EndSizePixels}`,
  `MaxWorldPositionOffsetDisplacement`)
- `materialLayers` — flattened `FMaterialLayersFunctions` (layers,
  blends, layer states, layer names, layer link keys) when present
- `parentLightmassSettings` — overridden Lightmass settings
  (`bOverrideCastShadowAsMasked`, etc.) and resolved values
- `nanitePassthrough` — `bIsNaniteOverride`, `NaniteOverrideMaterial`
- `physMaterial` / `physMaterialMask` overrides
- `subsurfaceProfile` override
- `shadingModelOverride` (already partially in BasePropertyOverrides
  string; lift to structured)
- `flags`: `bHasStaticPermutationResource`, etc.

Use the same code paths already exposed by
`material.authoring.get_material_instance_info`
(`F-material-instance-overrides-incomplete`, DONE) so dump output and live
read output stay byte-aligned. If the read fails, still write
`meta.json` / `properties.json` and record the failure via the existing
aspect-diagnostic channel.

## Fix

1. Add an explicit `else if (UMaterialInstanceConstant* MIC = Cast<UMaterialInstanceConstant>(Asset))`
   branch in `AssetDumpHandler.cpp` alongside the existing `UMaterial` /
   `UMaterialFunction` branches (around L446–455).
2. Extract a shared `MaterialInstanceDumpBuilder::BuildMaterialInstanceJson`
   helper (the live `material.authoring.get_material_instance_info` handler
   currently builds its payload inline at `MaterialAuthoringHandler.cpp:1971-2127`;
   this extraction is the prerequisite for the dump branch). Wire both call
   sites at the new helper so dump output and live read output stay
   byte-aligned. The RPC handler keeps its `inherited` + `parameters[]`
   enrichment on top of the shared subset.
3. Emit it as `DumpFileNames::MaterialInstance` (new constant in
   `AssetDumpHandler.h` — value `material_instance.json`).
4. Treat MIC failure the same way as Material/Function failures —
   preserve `meta.json` / `properties.json`, record diagnostic via
   `RecordAspectDiagnostic`.
5. Update `asset.dump` doc / wiki aspect list to include
   `material_instance.json`.
6. Update `CLAUDE.local.md` asset-dump kind enumeration (already
   mentions `material_instance.json` — confirm it now actually exists).
7. Regression test on at least one MIC: dumped folder must contain
   `material_instance.json` with the documented shape; structured
   `basePropertyOverrides.OpacityMaskClipValue` (etc.) must round-trip
   correctly when the parent material has `dithered_lod_transition` /
   `opacity_mask_clip_value` overrides set.

### Expected sidecar shape (stub)

```json
{
    "parent": "/Game/Path/M_ParentMaster",
    "parentChain": ["/Game/Path/MI_Intermediate", "/Game/Path/M_ParentMaster"],
    "overrides": {
        "scalar":  { "Roughness": 0.9 },
        "vector":  { "Tint": { "R": 1.0, "G": 0.5, "B": 0.2, "A": 1.0 } },
        "texture": { "BaseColor": "/Game/Tex/T_Carbon" },
        "staticSwitch": { "UseDetail": true },
        "staticComponentMask": {
            "ChannelMask": { "R": true, "G": false, "B": false, "A": false }
        }
    },
    "basePropertyOverrides": {
        "bOverride_BlendMode": true,
        "BlendMode": "BLEND_Masked",
        "bOverride_ShadingModel": true,
        "ShadingModel": "MSM_FromMaterialExpression",
        "bOverride_OpacityMaskClipValue": true,
        "OpacityMaskClipValue": 0.3333,
        "bOverride_TwoSided": false,
        "TwoSided": false,
        "bOverride_DitheredLODTransition": false,
        "DitheredLODTransition": false,
        "DisplacementScaling": { "Magnitude": 4.0, "Center": 0.5 },
        "DisplacementFadeRange": { "StartSizePixels": 4.0, "EndSizePixels": 1.0 },
        "MaxWorldPositionOffsetDisplacement": 0.0
    },
    "materialLayers": {
        "Layers":      ["/Game/MatLayers/ML_Rock", "/Game/MatLayers/ML_Moss"],
        "Blends":      ["/Game/MatLayers/MB_HeightBlend"],
        "LayerStates": [true, true],
        "LayerNames":  ["Background", "Moss"]
    },
    "parentLightmassSettings": {
        "bOverrideCastShadowAsMasked": false,
        "CastShadowAsMasked": false
    },
    "nanitePassthrough": {
        "bIsNaniteOverride": false,
        "NaniteOverrideMaterial": null
    },
    "physMaterial": null,
    "subsurfaceProfile": null,
    "flags": {
        "bHasStaticPermutationResource": true
    }
}
```

## Impact

- ~811+ MIC asset folders today carry zero structured override data.
- Static-switch overrides — the most common authoring lever for
  per-instance feature toggles — are completely opaque (buried inside
  the `EditorOnlyData.StaticParameters` export-text string).
- BasePropertyOverrides — opacity-mask clip, blend mode, displacement —
  require regex-scraping a comma-soup string.
- Material-instance diffs (live vs. baseline) only catch wholesale
  string changes, not field-level overrides, making MIC-driven content
  drift invisible to consumers.

## History
- `#1-initial-repro` `OPEN` reporter — Confirmed via three live samples (`MI_CarbonFiber3`, `MI_Old_Concrete_Barrier_vksrdes_8K`, `MI_Sharp_Cliff_rcygn_1K`) and a grep of `AssetDumpHandler.cpp` that no `UMaterialInstanceConstant` branch exists — MICs fall through to the generic `meta.json` + `properties.json` path. `REGISTER_DECOMPILE_IR` only registers `mgir.material` / `mgir.material_function`, not an MIC variant. `CLAUDE.local.md` lists `material_instance.json` as canonical so this is a handler gap, not a doc gap. ~811+ MICs in the App/ dump alone are affected.
- `#2-mic-builder-extracted` `IN-REVIEW` developer — Extracted `MaterialInstanceDumpBuilder::BuildMaterialInstanceJson` to share the structured payload between `asset.dump` and `material.authoring.get_material_instance_info`. Added MIC dispatch branch in `AssetDumpHandler.cpp`, `DumpFileNames::MaterialInstance` constant, `FixedCanonical[]` entry, refactored the RPC handler to call the builder for the shared subset (parent/overrides/basePropertyOverrides/materialLayers/etc.) while preserving the RPC-only `inherited` + `parameters[]` enrichment, and added regression test `FAssetDumpMaterialInstanceSidecarTest` exercising the dump file presence and structured field values via an in-memory MIC fixture.
- `#3-skip-mcp-offline` `SKIP` tester — MCP HTTP endpoint at 127.0.0.1:19880 is not listening (connection refused); editor process (pid 14872) is alive but `EditorAutomationRpcGatewaySubsystem deinitializing` was logged 2026-05-19 12:16:25 and the new DLL (built 2026-05-21 11:14:10 with the MIC branch / builder / `DumpFileNames::MaterialInstance` / `FixedCanonical[]` entry / regression test all present in source) was never loaded into the running editor. Cannot exercise `asset.dump` live without an editor restart, which the verify protocol forbids. Code-side claims confirmed via grep in `AssetDumpHandler.cpp:23,502-516,840`, `AssetDumpHandler.h:36`, `MaterialInstanceDumpBuilder.{h,cpp}`, `MaterialAuthoringHandler.cpp`, and `TestAssetDumpMaterialInstance.cpp`.
- `#4-verify-fix` `DONE` tester — Verified live: ran `asset.dump` on `/Game/UI/Menu/Art/MI_UI_MenuButton_Base` (substitute MIC, original repro `MI_CarbonFiber3` missing on disk). `writtenPaths` includes `material_instance.json` alongside `meta.json` / `properties.json`. Sidecar at `.editor-automation/asset-dumps/Game/UI/Menu/Art/MI_UI_MenuButton_Base/material_instance.json` contains all documented top-level keys: `parent` (`M_UI_Base_BordersAndButtons`), `parentChain`, `overrides` with structured `scalar` / `vector` (RGBA quads) / `texture` / `staticSwitch` (`IsButtonBorder: true`, etc.) / `staticComponentMask` (per-channel bool quads), `basePropertyOverrides` with all `bOverride_*` flags + per-field values (`BlendMode: BLEND_Translucent`, `OpacityMaskClipValue: 0.3333`, `DisplacementScaling.{Magnitude,Center}`, `DisplacementFadeRange.{StartSizePixels,EndSizePixels}`), `nanitePassthrough`, `physMaterial` / `physMaterialMask` / `subsurfaceProfile`, `flags.bHasStaticPermutationResource`.
