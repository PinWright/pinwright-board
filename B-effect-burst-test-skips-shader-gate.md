---
id: B-effect-burst-test-skips-shader-gate
title: "ShortBurstAtEarlySampleIsVisible skips on every UE 5.8 real-RHI run: its shader gate only reads the map after capture, and on 5.5+ editor shader maps are never complete unless a full compile is requested"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, effect, niagara, step_and_capture, shader-compile, tests-skip, test-fixture, notCompiled]
encounters: 1
lastSeen: 2026-09-29T16:54:20Z
---

# ShortBurstAtEarlySampleIsVisible never reaches its pixel assertions on 5.8

`PinWright.effect.step_and_capture.ShortBurstAtEarlySampleIsVisible` (`Tests/Niagara/TestEffectStepAndCapture.cpp`) skipped in an offscreen D3D11 SM5 full suite with `reason=shader-compile-unavailable -- niagara renderer material shader map not compiled on this host: /Niagara/DefaultAssets/DefaultSpriteMaterial.DefaultSpriteMaterial status=notCompiled, /Niagara/DefaultAssets/M_Gnomon_Alpha.M_Gnomon_Alpha status=notCompiled` (`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:28937`). The three captures (warm-up, baseline, burst) had all succeeded; the whole test took 0.43 s. The only behavioural test of `effect.step_and_capture` (see `F-effect-step-and-capture-atomic`) therefore measured nothing.

**Root cause (test).** The gate added in `F-effect-step-and-capture-atomic` `#9` read each renderer material with `MaterialShaderState::ProbeAfterCapture` after the captures. That helper never submits a compile (`Handlers/Material/MaterialShaderState.h:452-483`); it only waits while `IsCompilationFinished()` is false, then classifies `IsGameThreadShaderMapComplete()`. On UE 5.5+ `r.ShaderCompiler.JobCacheDDC` defaults to true (`ShaderCompilerJobCache.cpp:77-81` on 5.8: "Skips compilation of all shaders on Material and Material Instance PostLoad and relies on on-demand shader compilation"), so `IsMaterialMapDDCEnabled()` is false, PostLoad caches with `EMaterialShaderPrecompileMode::None`, and the renderer compiles single permutations on demand (`FMaterial::TryGetShaders`, `MaterialShared.cpp:3963-4250`). The game-thread map is never complete, so the probe reads `notCompiled` forever on a host that renders the materials fine. The skip was deterministic, not a host limitation.

The mis-classification in the shared probe is its own root cause, recorded on `B-material-readiness-false-notcompiled-warning` `#4`.

## History

- `#1-skips-on-real-rhi` `OPEN` reporter — Offscreen full suite `499a9295` (D3D11 SM5, RTX 4060) skipped this test with `shader-compile-unavailable` after all three captures succeeded (`automation.log:28928-28938`). No shader-compile error or ShaderCompileWorker failure precedes the skip. Traced to the gate reading `ProbeAfterCapture`, which cannot compile anything, against shader maps that the 5.5+ on-demand compile model never completes.
- `#2-force-compile-before-capture` `IN-REVIEW` developer — `Tests/Niagara/TestEffectStepAndCapture.cpp`: the renderer-material enumeration moved ahead of the first capture, and each material is compiled with the existing `PinWright::MaterialShaderState::ProbeAndWait` (`CacheShaders(Synchronous)` submits every remaining permutation, then `MaterialCompileErrorCollector::WaitAndCollect` drains it under its bounded 90 s pumping wait). The test skips (`shader-compile-unavailable`) only when `GShaderCompilingManager` is absent or `IsShaderCompilationSkipped()`; the NullRHI case is already skipped earlier by `SkipIfRenderingUnavailable`. Any other non-`completed` status (`failed`, `timedOut`, `outstanding`, `notCompiled`) is now an `AddError` naming each material, its status and seconds waited. The post-capture `ProbeAfterCapture` gate and its skip were removed, and the orphaned `HAL/PlatformTime.h` include with it; `ShaderCompiler.h` is included for `GShaderCompilingManager`. Pixel assertions are unchanged. Compile-checked with UBT `-SingleFile` on UE 5.8 (`Result: Succeeded`); automation not run. Known cost: the first run on a cold per-shader DDC compiles every permutation of both engine materials (bounded by the 90 s wait per material, except a map that has to be built from scratch, which `CacheShaders(Synchronous)` compiles inline).
