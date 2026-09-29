---
id: B-thumbnail-mi-test-probes-without-wait
title: "asset.generate_thumbnail.MaterialInstanceFallbackRequiresOptIn reads its broken instance's shader state with a non-blocking Probe, sees 'outstanding', and skips"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, testing, material, shader-compile, thumbnail, fixture-skip]
encounters: 1
lastSeen: 2026-09-29T17:16:39Z
---

# Material-instance fallback test does not wait for its fixture's compile

`PinWright.asset.generate_thumbnail.MaterialInstanceFallbackRequiresOptIn`
(`Tests/Render/TestCaptureAssetPreviewMaterialFallback.cpp`) skips with
`reason=shader-compile-unavailable -- The broken instance permutation reported 'outstanding'`
(suite log `Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:12758`; also in run `8579e054`).

The fixture instance gets its own static permutation (`UseBrokenHlsl` switch) and
`ForceRecompileForRendering(Synchronous)`, but the test then reads the state with
`MaterialShaderState::Probe`, which is a snapshot: the permutation's result is not finalised until
`ProcessAsyncResults` runs on the game thread, so `FMaterial::IsCompilationFinished()` is still false
and the probe answers `outstanding`. Not a host limitation: the sibling tests on plain materials use
`ProbeAndWait` (`PWMtlFallbackRequireBrokenMaterial`) and run.

## History
- `#1-probe-snapshot-reads-outstanding` `OPEN` reporter — The instance permutation is probed without a wait, so the compile the fixture just submitted reads `outstanding` and the fallback-policy assertions never run.
- `#2-probe-and-wait` `IN-REVIEW` developer — Replaced `Probe(Instance)` with `MaterialShaderState::ProbeAndWait(Instance)` (bounded, game-thread-pumping drain via `MaterialCompileErrorCollector::WaitAndCollect`, which compiles the instance itself because it owns a static permutation). The `shader-compile-unavailable` skip remains only for a status other than `failed` after the wait. `-SingleFile` compile succeeded; not run.
