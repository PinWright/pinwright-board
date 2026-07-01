---
id: B-material-stats-instruction-count-hardcoded
title: "asset.get_material_stats `instructionCount` is hardcoded to -1 — never computed, so the caller's complexity/compile check always reads a sentinel that looks like a real count"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, material, get_material_stats, instructionCount, hardcoded, stub, silent-wrong-data, AssetMaterialHandler]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `asset.get_material_stats` always returns `instructionCount: -1`

`asset.get_material_stats` is documented as "Get material statistics (shading
model, samplers, etc.)" and returns `{shadingModel, instructionCount,
samplerCount}`. The `shadingModel` and `samplerCount` fields are computed for
real (samplerCount counts the `TextureSample` expressions), but
**`instructionCount` is hardcoded to `-1` and never computed**. The handler
declares a local `int32 InstructionCount = -1;` and writes it straight into the
result with no attempt to query the compiled shader's instruction count.

This is silent wrong/hardcoded data on a normal-path typed getter. The field is
named `instructionCount` and sits in a `stats` object next to two genuine
statistics, so the value reads as a real number. A caller using this verb to
"confirm the graph compiled" or to gauge material complexity (the exact intent
that surfaced this — building a master material and then pulling stats to verify
it compiled) gets `-1` regardless of how trivial or how heavy the graph is. There
is no `instructionCount: null`, no "unavailable" marker, and no error — just a
sentinel that the caller trusts. `-1` is also a plausible-looking value a naive
caller might compare with `> 0` or print verbatim.

## What it should do

Either compute the real value or signal that it is unavailable, instead of
emitting a fixed `-1` dressed up as a statistic. UE exposes the compiled
instruction count via the material's shader map stats
(`FMaterialResource::GetUserInterpolatorUsage` / the shader-stats path used by the
Material Editor's "Stats" overlay, e.g. `UMaterial::GetMaterialResource(...)
->GetCompileErrors()` / `GetRepresentativeInstructionCounts(...)` or
`FMaterialStatsUtils`). If computing it is out of scope, drop the field or return
`null`/omit it so the response does not promise a count it never produces; do not
ship a fixed `-1` as if it were measured.

**Repro** (verbatim, replayed via `mcp__editor-automation__call`):
- `asset.create_material` `{name:"M_TintedFloor", path:"/Game/FuzzMaterials",
  properties:{ShadingModel:"DefaultLit", BlendMode:"Opaque"}}` -> success
- add Constant3Vector (idx 0), Constant (idx 1), Multiply (idx 2);
  connect 0->2.A and 1->2.B; add scalar param Roughness; `asset.save force:true`
- `asset.get_material_stats` `{assetPath:"/Game/FuzzMaterials/M_TintedFloor"}` ->
  `{"success":true,"stats":{"shadingModel":"DefaultLit","instructionCount":-1,"samplerCount":0}}`

The `-1` is unconditional: the handler at
`Source/PinWright/Private/Handlers/Asset/AssetMaterialHandler.cpp`
(the `asset.get_material_stats` block) sets `int32 InstructionCount = -1;` then
`Stats->SetNumberField(TEXT("instructionCount"), InstructionCount);` with no
intervening computation — so every material, of any complexity, reports `-1`.

**Workaround:** none for the value itself — there is no sibling RPC that returns a
material's instruction count. Treat `instructionCount` as meaningless and ignore it.

**Fix:** compute the instruction count from the material's compiled shader-map
stats and emit the real number; if that path is unavailable for the asset, emit
`null` / omit the field rather than a fixed `-1`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed the full M_TintedFloor build via mcp__editor-automation__call: `asset.get_material_stats` returns `{"shadingModel":"DefaultLit","instructionCount":-1,"samplerCount":0}`. Confirmed in source: `AssetMaterialHandler.cpp` `asset.get_material_stats` hardcodes `int32 InstructionCount = -1;` and writes it directly into the `stats` object — never computed. samplerCount/shadingModel are real; instructionCount is a fixed sentinel masquerading as a statistic on a normal-path typed getter.
- `#2-fix` `IN-REVIEW` developer — Replaced the fabricated `int32 InstructionCount = -1; SetNumberField(...)` at `AssetMaterialHandler.cpp:469-470` with an honest `Stats->SetField("instructionCount", MakeShared<FJsonValueNull>())` so the field no longer ships a sentinel that reads as a measured count. Chose the ticket's null fallback over computing the real value because the engine's only instruction-count utility, `FMaterialStatsUtils::GetRepresentativeInstructionCounts`, is not DLL-exported from MaterialEditor (no `MATERIALEDITOR_API`) and its one exported wrapper `ExtractMatertialStatsInfo` takes the module-private `FShaderStatsInfo` type — neither is callable cross-module — and the count is only populated once the offline platform shader compiler has run, which the headless automation path never triggers; reimplementing the render-internal representative-shader walk is version-fragile across UE 5.3–5.7. Added `#include "Dom/JsonValue.h"`. Files: `Source/PinWright/Private/Handlers/Asset/AssetMaterialHandler.cpp`. Regression test `FAssetGetMaterialStatsInstructionCountNotHardcodedTest` (`PinWright.asset.get_material_stats.InstructionCountNotHardcoded`) in `Source/PinWright/Private/Tests/Assets/TestAssetHandlers.cpp` builds a real scratch UMaterial, calls the handler, and asserts `stats.instructionCount` is `EJson::Null` (not a number) while `shadingModel` is still present — it fails if the fix reverts to `SetNumberField(-1)`.
