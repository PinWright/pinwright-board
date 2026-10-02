---
id: B-compile-mgir-shadercompile-block-missing
title: "material.compile_mgir returns no shaderCompile block at all (even with waitForShaderCompile:true), although the method page and material.mgir say the verdict lives there"
status: DONE
severity: Medium
category: bug
tags: [material, mgir, compile_mgir, shadercompile, response-shape, docs-mismatch]
encounters: 1
lastSeen: 2026-09-29T18:40:00Z
duplicateOf: E-material-verbs-have-no-shader-compile-signal
---

# `material.compile_mgir` omits the documented `shaderCompile` block

## Symptom

`material.compile_mgir` is documented (method page and `material.mgir` § Compile / decompile) to
return a `shaderCompile` block — "branch on `shaderCompile.status`", "pass
`waitForShaderCompile: true` ... and report the real verdict", with per-asset
`shaderCompile.materials[]` and failures raised into `warnings[]`.

The live response carries no such key. Observed key set (PDS unreal-fpv checkout, UE 5.8 Linux,
editor not in PIE, `waitForShaderCompile: true`, `save: false`, `mode: "Append"`, target
`/App/MultiplayerLevelEditor/PostProcess/M_SelOverlay_O1_k005`, a translucent Modulate surface
material with two Custom HLSL nodes):

```
['assetPaths', 'blocksCompiled', 'consumerRefresh', 'expressionsCreated', 'mode',
 'saveDetail', 'saveRequested', 'saveState', 'saved', 'world']
```

Earlier in the same session, six other `compile_mgir` calls (post-process material with a Custom
node, four new surface materials; `save: true`, with and without `waitForShaderCompile: true`,
some while PIE was active → `saveState: blockedByPie`) also returned `shaderCompile: None` /
no `warnings`.

## Impact

The docs tell the caller that `blocksCompiled` is not a shader verdict and that `shaderCompile`
is the measurement. With the block missing, a caller has no shader verdict from this verb and
must add a separate `material.authoring.compile_material` per asset (which did return
`compileStatus: completed`, `compileErrors: []` for all of them). A caller that reads
`r.get('shaderCompile', {}).get('status')` gets `None` and may treat it as "not compiled" or
skip the check.

## Repro

1. Any `material.compile_mgir` call with `waitForShaderCompile: true` on a small material.
2. Inspect the response keys: no `shaderCompile`.

## Expected

Either the `shaderCompile` block as documented (status + per-material breakdown), or the docs
updated to point callers at `material.authoring.compile_material`.

## History

- `#1-filed-missing-block` `OPEN` reporter — Found while prototyping map-editor selection outline/overlay materials via MGIR on the PDS unreal-fpv checkout. Worked around with a follow-up `material.authoring.compile_material` per asset.
- `#2-fixed-by-fold-fix` `IN-REVIEW` developer — Same defect as `E-material-verbs-have-no-shader-compile-signal` `#3`/`#4` (`#4` is this very session's MPP_SelectionOutline call); no new code needed. Root cause: `compile_mgir` folds each written material through `FState::Accumulate` (`Source/PinWright/Private/Handlers/Material/MaterialShaderState.h`), whose default status `NotMeasured` outranked `completed`/`onDemand` under worst-wins, so an all-healthy document stayed `NotMeasured` and `AddReport` emitted no `shaderCompile` block — exactly the observed key set (the reporter's materials compiled cleanly, as the follow-up `compile_material` showed). Fixed by plugin commit `29e9d445` (first fold takes the material's status outright), which landed about an hour after this ticket's `lastSeen`, so the reporting build predates it. Verb-level regression tests landed in `3590aa9c`, both ancestors of the current submodule HEAD `9bb70b90`: `Source/PinWright/Private/Tests/Material/TestCompileMgirShaderCompileReport.cpp` — `PinWright.material.compile_mgir.shader_compile.FoldPublishesHealthyVerdict` (pure fold logic, fails deterministically on the pre-fix fold), `PinWright.material.compile_mgir.shader_compile.CleanMaterialReportsCompletedWithAndWithoutSave` and `PinWright.material.compile_mgir.shader_compile.BrokenHlslReportsErrorsWithAndWithoutSave` (invoke `material.compile_mgir` through the handler with `waitForShaderCompile:true`, `save:false`/`true`; block presence asserted unconditionally, `materials[]` emitted by `AddReport` for every fold). Docs (`material.mgir`, `material.compile-state`, method description) already match the fixed behaviour. Filter: `PinWright.material.compile_mgir.shader_compile`. Verify by re-running the repro on a build at or after `29e9d445`.
- `#3-verified-linux` `DONE` tester — PinWright `10212ee4` (on origin/master `6283b63b`), UE 5.8 Linux Vulkan. Runs: w23-final = offscreen full suite, no DISPLAY, 5568/5568 passed; w23-xfinal = DISPLAY=:0 offscreen, drive.os_input+click_occlusion+os_gesture+input, 31/31; w23-vis = DISPLAY=:0 windowed drive.input.ModifierChord, 2/2; Python = Content/Python/tests, 427 OK / 5 skipped (all skips Windows-only or an absent local script). All three `PinWright.material.compile_mgir.shader_compile.*` tests passed in w23-final with no skip: `FoldPublishesHealthyVerdict` (pure fold), `CleanMaterialReportsCompletedWithAndWithoutSave` and `BrokenHlslReportsErrorsWithAndWithoutSave` (real `material.compile_mgir` with `waitForShaderCompile:true`, `save:false` and `true`; the `shaderCompile` block is asserted present unconditionally; clean -> completed with no warnings; broken HLSL -> failed with errors and a `warnings[]` entry). Expected (the documented block is returned) is met on the current tree. The duplicate target `E-material-verbs-have-no-shader-compile-signal` was outside this round's scope and was not touched.
