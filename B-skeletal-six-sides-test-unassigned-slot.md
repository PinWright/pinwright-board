---
id: B-skeletal-six-sides-test-unassigned-slot
title: "render.capture_asset_preview.SkeletalMeshAssetGetsSixSides can never measure: its SkeletalCube fixture has an unassigned material slot, so the verb correctly returns MATERIAL_FALLBACK"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, render, capture, material-fallback, silent-skip, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:27:48Z
---

# SkeletalMeshAssetGetsSixSides always skips

`PinWright.render.capture_asset_preview.SkeletalMeshAssetGetsSixSides`
(`Tests/Render/TestAssetPreviewSubjects.cpp`) skipped `reason=capture-unavailable` with
`MATERIAL_FALLBACK` (`Saved/Logs/pw_gapwave_full_offscreen2.log:45983`).

Not shader-compile state and not an environment condition. The fixture
`/Engine/EngineMeshes/SkeletalCube` imports no material at all (its package imports only
`SkeletalCube_Skeleton`), so its rendered slot is unassigned. `MaterialShaderState::ProbeCaptureAsset`
records it via `AddUnassignedSubject`, `AddCaptureReadiness` reports `unassignedMaterialSlot`, and
`ApplyCaptureFallbackPolicy` refuses without `allowFallback` (`RenderHandler.cpp` MATERIAL_FALLBACK
branch). That is the verb's documented contract; incomplete shader maps are already a warned
success. The test therefore skips deterministically on every host and its shot-count, captureSource
and subject assertions never run.

**Fix:** the test passes `allowFallback: true`; the fixture's missing material is not what it
measures.

## History
- `#1-deterministic-fallback-skip` `OPEN` reporter — Filed from the full offscreen suite: the skip is deterministic (fixture has no material), so the six-sides assertions never ran on any host.
- `#2-allow-fallback-in-test` `IN-REVIEW` developer — `Tests/Render/TestAssetPreviewSubjects.cpp` `SkeletalMeshAssetGetsSixSides` now passes `allowFallback: true`, so the capture is kept with the verb's unassigned-slot warning and the six-shot, captureSource, subject and poseSet assertions run. No handler change: MATERIAL_FALLBACK here is the verb's correct answer. `-SingleFile` compile: clean.
