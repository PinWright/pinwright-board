---
id: B-material-compile-wait-exits-before-ondemand-jobs
title: "Material compile wait returns 'outstanding' without timing out: exits before render-thread on-demand jobs register"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, material, shader-compile, race, tests-skip]
---

# Material compile wait returns 'outstanding' without timing out: exits before render-thread on-demand jobs register

`MaterialCompileErrorCollector::WaitAndCollect` (used by `MaterialShaderState::ProbeAndWait`, `material.authoring.compile_material`, MGIR `waitForShaderCompile`) loops while `!Resource->IsCompilationFinished()`, then measures `bOutstanding = !Resource->IsCompilationFinished()`. It returned `outstanding` with `bTimedOut=false` for a material instance with its own static permutation: `PinWright.asset.generate_thumbnail.MaterialInstanceFallbackRequiresOptIn` skipped with `The broken instance permutation reported 'outstanding'`. The whole test took 0.6 s (`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:12736-12760`), so the 90 s timeout never fired. The instance's permutation errors (`FDebugViewModePS`, base-pass PS) were logged in the same window, so those jobs were running.

Root cause (UE 5.8 source): the `FMaterial::CacheShaders` completion computes `bRequiredComplete = !bMaterialInstance && IsRequiredComplete()` (`MaterialShared.cpp` ~3127). An instance that already has a game-thread shader map keeps it even when it is incomplete, so `CacheShaders(Synchronous)` submits nothing for the missing permutations. The render thread submits them on demand (`FMaterial::TryGetShaders`) for frames already in flight. `IsCompilationFinished()` (`MaterialShared.cpp:876-890`) checks only whether jobs are registered for the compiling shader-map id. The game-thread poll can therefore read "finished" before the render thread registers the jobs, and the read immediately after the loop sees them. The wait's contract, "done, or timed out, and says which", is broken, and callers publish `outstanding` as though no wait had happened.

**Fix:** a "finished" read is accepted only after it survives `FlushRenderingCommands()` plus a game-thread pump. Once flushed, the render thread cannot submit more work until the game thread ticks again, and the wait holds the game thread. The flush is skipped when `PinWrightSafePoint::IsSafeNow()` is false: a flush inside a tick or a named-thread pump is a documented deadlock source (`Dispatch/SafePoint.cpp`). In that position the wait falls back to the pre-fix single read.

## History
- `#1-outstanding-without-timeout` `OPEN` reporter — `ProbeAndWait` on a broken static-permutation material instance returned `outstanding` without timing out in the 5.8 offscreen suite, so `asset.generate_thumbnail.MaterialInstanceFallbackRequiresOptIn` skipped its assertions. Diagnosis above.
- `#2-flush-before-trusting-finished` `IN-REVIEW` developer — `Handlers/Material/MaterialCompileErrorCollector.h` (header; `WaitAndCollect` is inline there): the drain loop now returns on "finished" only after `FlushRenderingCommands()` + `AdvanceOnGameThread()` re-reads finished, gated on `PinWrightSafePoint::IsSafeNow()`; includes `RenderingThread.h` and `Dispatch/SafePoint.h`. Failure direction: `TestCaptureAssetPreviewMaterialFallback.cpp` `MaterialInstanceFallbackRequiresOptIn` now fails with `AddError` when `ProbeAndWait` returns `outstanding`, which the contract forbids, where it used to skip. The race depends on render-thread timing, so no fully deterministic reproduction exists. The test fires whenever the race occurs, and it did occur in run 499a9295. Not built or run.
