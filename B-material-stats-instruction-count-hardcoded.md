---
id: B-material-stats-instruction-count-hardcoded
title: "asset.get_material_stats `instructionCount` is hardcoded to -1 — never computed, so the caller's complexity/compile check always reads a sentinel that looks like a real count"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, material, get_material_stats, instructionCount, hardcoded, stub, silent-wrong-data, AssetMaterialHandler]
encounters: 2
lastSeen: 2026-09-03T07:20:00+03:00
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
- `#3-fix-premise-is-wrong-on-58` `IN-REVIEW` reporter — Hit the `null` on `/Game/FPS/Weapons/Materials/M_WPN_Master` (returns `{"shadingModel":"DefaultLit","instructionCount":null,"samplerCount":9}`). The `null` is honest and is an improvement on `-1`, so this entry is not a re-report of the original defect — it is a correction to `#2-fix`'s reason for not computing the value. **That reasoning does not hold on UE 5.8.** `#2` deferred on the grounds that "the engine's only instruction-count utility, `FMaterialStatsUtils::GetRepresentativeInstructionCounts`, is not DLL-exported from MaterialEditor ... neither is callable cross-module". There is a second, fully exported route it did not consider: `UMaterialEditingLibrary::GetStatistics(UMaterialInterface*)`, declared `static UE_API FMaterialStatistics GetStatistics(...)` at `Engine/Source/Editor/MaterialEditor/Public/MaterialEditingLibrary.h:610` — a **public** header, `UE_API` expanding to `MATERIALEDITOR_API`. It returns the `USTRUCT FMaterialStatistics` (same header, L21-46) carrying `NumVertexShaderInstructions`, `NumPixelShaderInstructions`, `NumSamplers`, `NumVertexTextureSamples`, `NumPixelTextureSamples`, `NumVirtualTextureSamples`, all `BlueprintReadWrite`. Verified live through the Python binding, which is the same C++ function: `unreal.MaterialEditingLibrary.get_statistics(m)` on M_WPN_Master returned `num_vertex_shader_instructions: 156, num_pixel_shader_instructions: 441, num_samplers: 12, num_pixel_texture_samples: 12` — real, varying numbers, not sentinels. They also respond correctly to graph changes: toggling one `StaticSwitchParameter` that gates a World Position Offset chain moved the vertex count 156 -> 185 and left the pixel count at 441, and bypassing a 5-node cavity chain moved the pixel count 441 -> 437. So the counts are both available headlessly and sensitive enough to be worth reporting. Suggested fix: call `UMaterialEditingLibrary::GetStatistics` and emit the real numbers, ideally widening the response past a single `instructionCount` to the vertex/pixel split the struct already provides (a single scalar cannot represent a material whose vertex cost changed and pixel cost did not — exactly the WPO case above). Keep `null` only as the fallback when no shader map exists. Caveat carried over from `#2` and still true: the counts are populated only once a shader map is present, so callers should run `material.authoring.compile_material` first — in this session an uncompiled read and a compiled read differed. Workaround until fixed: `python.execute` with `unreal.MaterialEditingLibrary.get_statistics`.
- `#4-null-survives-a-successful-compile-and-the-docs-still-point-here` `IN-REVIEW` reporter — **Corroborates `#3` and adds two things it does not have: the `null` survives the exact workflow the wiki prescribes, and the source comment still states `#2`'s disproved premise as fact.** Same defect, same material as `#3`, so `encounters` / `lastSeen` are left alone on `#3`'s precedent. **Measured this session on `/Game/FPS/Weapons/Materials/M_WPN_Master`, three `asset.get_material_stats` calls at three graph sizes:** at ~95 expressions, `{"success":true,"stats":{"shadingModel":"DefaultLit","instructionCount":null,"samplerCount":9}}`; after adding 45 expressions, `... "instructionCount":null,"samplerCount":10}`; final at 152 nodes, `... "instructionCount":null,"samplerCount":11}`. **`samplerCount` tracked the edits correctly, 9 -> 10 -> 11, across those same three calls** — so the handler is reading something real and only `instructionCount` is dead, which makes the null read less like "no data yet" and more like a property of the material. **The null is unconditional, confirmed in the current source:** `Handlers/Asset/AssetMaterialHandler.cpp:259` is a bare `Stats->SetField(TEXT("instructionCount"), MakeShared<FJsonValueNull>());` — no branch, no shader-map probe. **`#3`'s "compile first" caveat therefore does not rescue it, and that was measured rather than inferred:** the third reading above was taken immediately after `material.authoring.compile_material` returned `compileStatus: "completed"`, `compileSucceeded: true`, `shaderCompile.status: "completed"` — i.e. with a finished shader map present, exactly the state the docs tell you to reach before reading stats — and the field was still `null`. **No second route exists through the RPC surface:** `material.authoring.get_material_info` carries no instruction count in any form (the word "instruction" does not appear on `Saved/PinWright/wiki/material.authoring.get_material_info.md`), so `#3`'s `python.execute` + `unreal.MaterialEditingLibrary.get_statistics` is still the only way to obtain the number. **Documentation half, checked against the pages rather than recalled, and fixable ahead of the handler:** `Saved/PinWright/wiki/asset.get_material_stats.md` is twelve lines, describes the verb only as "Get material statistics (shading model, samplers, etc.)", and never names `instructionCount` at all, let alone that it is always null; and `Saved/PinWright/wiki/material.authoring.compile_material.md:21` actively routes callers here — "Material-derived stats (use `call("asset.get_material_stats", ...)` after compile)." A caller who follows that instruction gets a null and no explanation of it. **One more thing a fixer will trip on:** the comment at `AssetMaterialHandler.cpp:249-258` still states `#2`'s reasoning as fact — that `FMaterialStatsUtils::GetRepresentativeInstructionCounts` is the engine's only route and is not callable from this module, and that the count "is also only populated once the offline platform shader compiler has run, which does not happen on the headless automation path". `#3` disproved both on UE 5.8 with live, varying numbers through the exported `UMaterialEditingLibrary::GetStatistics`. Correct that comment as part of the fix, or the next reader re-derives the wrong conclusion straight from the source. **Why this outlives one run:** instruction count is the standard budget number for a shader edit, and with the field dead I had to hand-estimate ~75-80 added instructions for a material I had just authored — an unmeasured number standing in for a measured one, which is the class of thing this board exists to stop.
