---
id: B-compile-material-false-shader-success
title: "compile_material reports compiled:true while shader permutations fail to compile"
status: DONE
severity: High
category: bug
tags: [material, material-authoring, compile, shaders, silent-failure, false-success]
---

# compile_material reports compiled:true while shader permutations fail

`material.authoring.compile_material` returns `{compiled:true, saved:true}` even
when the material's shader permutations fail to compile, so the asset silently
falls back to the Default Material in-game. The handler
(`MaterialAuthoringHandler.cpp` lines 2520–2544) only calls
`Material->PreEditChange(nullptr)` + `PostEditChange()` + `MarkPackageDirty()`,
optionally saves, then **unconditionally** sets `compiled:true` (line 2540). It
never awaits the shader compile (`GShaderCompilingManager->FinishCompilation`),
inspects the `FMaterialResource` compile errors, or checks the shader-compile
warning log. Per-VF/permutation HLSL errors are never surfaced. Agents trusting
the success payload ship a broken material.

This is distinct from `B-performance-optimize-shaders-no-completion-signal`
(that was a missing async completion signal) and from `B-false-compile-success`
(BPIR/blueprint compile). It is the same *family* as
`B-material-stub-handlers-silent-success` (false success on the material API),
but `compile_material` does real work — the failure is the missing error
reporting on the shader-compile step, not a no-op stub.

**Workaround:** Run `system.console_command {command:"recompileshaders material <Name>"}`
then grep `Saved/Logs/PDS.log` for `Failed to compile`.

**Fix:** Await the shader compile and report failures — add a
`compiledWithErrors` flag plus error text by reading `FMaterialResource` compile
errors (or the shader-compile warning log) after `PostEditChange()`; alternatively
route through `Ctx.StartJob()` + `GShaderCompilingManager` like
`performance.optimize_shaders` and report the final error state.

## History
- `#1-initial-repro` `OPEN` reporter — Called `material.authoring.compile_material {assetPath:"/App/Gates/Models/M_Text3DCylWrap"}` → returned `{"compiled":true,"saved":true}` repeatedly on a material whose WPO custom-HLSL node used `GetPrimitiveData(Parameters).LocalToWorld[2].xyz` (invalid — LocalToWorld is FDFMatrix). Shaders actually FAILED: `Saved/Logs/PDS.log` showed `LogShaderCompilers: Warning: Failed to compile Material /App/Gates/Models/M_Text3DCylWrap … Default Material will be used in game.` with `/Engine/Generated/Material.ush:3622: error: type 'FDFMatrix' does not provide a subscript operator` (X3121). Confirmed in source: `MaterialAuthoringHandler.cpp` lines 2520–2544 only do PreEditChange/PostEditChange/MarkPackageDirty (+optional save) then hardcode `compiled:true` (line 2540) with no shader-compile await or error check. The only way to detect the real failure was `recompileshaders material M_Text3DCylWrap` + grepping the log.
- `#2-await-and-report-shader-errors` `IN-REVIEW` developer — `compile_material` now blocks on shader compilation and surfaces failures. After PostEditChange the handler calls a new `MaterialCompileErrorCollector::WaitAndCollect` helper (`Private/Handlers/Material/MaterialCompileErrorCollector.h`) that runs `GShaderCompilingManager->FinishAllCompilation()` then reads `Material->GetMaterialResource(GMaxRHIFeatureLevel)->GetCompileErrors()` (after shader compile, `ProcessCompiledShaderMaps` writes failed-permutation HLSL errors back into the resource). Response gains `compiledWithErrors` (bool) and `compileErrors` (string[]); `compiled` stays `true` since compilation ran. Updated the handler description and the `wiki-src/material.authoring.md` overlay. Regression test `Private/Tests/Material/TestCompileMaterialShaderErrors.cpp` builds a sandbox material whose EmissiveColor is driven by a Custom node with the invalid `LocalToWorld[2]` HLSL, invokes the production handler, and asserts `compiledWithErrors:true` with a non-empty `compileErrors` array matching the subscript/FDFMatrix signature. Scope is exactly five files — `MaterialAuthoringHandler.cpp` (compile_material handler only), `MaterialCompileErrorCollector.h` (new), `TestCompileMaterialShaderErrors.cpp` (new), `wiki-src/material.authoring.md`, and this board file. The `MaterialHandlerUtils.h` param-aliasing refactor (E-material-editor-param-name-drift) and the `IsMaterialEditorOpen` open-editor guard (B-material-graph-edit-clobbered-by-open-editor, lives in `MaterialFinders.h` + `MaterialParameterCollectionHandler.cpp` + its own test) are NOT part of this ticket and were kept out of the authoring handler. Not yet compiled/run — pending build.
- `#3-verify-fix` `DONE` tester — Verified live. The repro asset `/App/Gates/Models/M_Text3DCylWrap` has since been repaired (custom HLSL now uses `DFToFloat3x3(...)` then subscripts the resulting `float3x3`, so `compile_material` correctly returns `compiledWithErrors:false`). To exercise the error path, created a sandbox material `/Game/App/UI/Test/M_McpVerifyTemp_compile_material_false_shader_success` (via `material.authoring.create_material` + `material.compile_mgir`) with a Custom node driving EmissiveColor whose Code is `GetPrimitiveData(Parameters).LocalToWorld[2].xyz` (the original invalid FDFMatrix subscript). `material.authoring.compile_material` on it returned `compiled:true, compiledWithErrors:true` with `compileErrors` containing `error: type 'FDFMatrix' does not provide a subscript operator` and `error X3121: ... indexable object type expected in index expression` — the exact signature from the acceptance criteria. Sandbox material deleted via `asset.delete`.
