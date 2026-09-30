---
id: B-zfight-depth-probe-vulkan-readback-crash
title: "render.detect_z_fighting crashes the editor on Vulkan: depth probe target is PF_R32_FLOAT, which Vulkan's FLinearColor readback check()-fails on"
status: IN-REVIEW
severity: Critical
category: bug
tags: [render, detect_z_fighting, scene-capture, vulkan, linux, readback, crash]
encounters: 1
costly: 1
lastSeen: 2026-09-30
---

# render.detect_z_fighting crashes the editor on Vulkan: depth probe target is PF_R32_FLOAT

Every `render.detect_z_fighting` call on a Linux Vulkan editor asserts and then SIGSEGVs:
`Assertion failed: false [VulkanRenderTarget.cpp:171] Unsupported format [100] for conversion to FLinearColor!`
via `FRenderTarget::ReadLinearColorPixels` -> `FVulkanDynamicRHI::RHIReadSurfaceData`.
The full suite dies at `PinWright.render.capture_vocabulary.ZFightingPublishesTheSceneCaptureValues`
(and the `render.detect_z_fighting.*` render fixtures crash the same way). Evidence:
`Saved/PinWright/test-runs/integrator_base_zf1/automation.log` lines 3609-3615 of the host project.

Cause: `FSceneCaptureProbe::EnsureRenderTarget` (`Source/PinWright/Private/Handlers/Render/SceneCaptureProbeUtils.cpp`)
created the `SCS_SceneDepth` target as `PF_R32_FLOAT` (VK_FORMAT_R32_SFLOAT = 100) and `Capture()` reads it with
`ReadLinearColorPixels`. Vulkan's `ConvertRawDataToFLinearColor` has no R32_SFLOAT case. D3D11 special-cases
R32F, so Windows runs never showed it.

**Fix:** depth target is now `PF_A32B32G32R32F` (full 32-bit float, depth in R), which Vulkan and the shared
`ConvertRAWSurfaceDataToFLinearColor` path (D3D11/D3D12/Metal) both convert. No other plugin readback uses an
unconvertible format (the 8-bit colour captures are BGRA8 via `ReadPixels`).

## History
- `#1-vulkan-r32f-readback-crash` `OPEN` reporter — Full suite crashes on Linux Vulkan at `render.capture_vocabulary.ZFightingPublishesTheSceneCaptureValues`; triaged to the PF_R32_FLOAT depth probe target read via ReadLinearColorPixels.
- `#2-depth-target-rgba32f` `IN-REVIEW` developer — Added `PinWrightSceneCaptureProbe::AnalysisTargetFormat(bool)` in `SceneCaptureProbeUtils.h/.cpp`; `EnsureRenderTarget` uses it, and it returns `PF_A32B32G32R32F` for depth on 5.4+ (was `PF_R32_FLOAT`), `PF_FloatRGBA` otherwise (5.3 branch unchanged). Regression test `PinWright.render.detect_z_fighting.ProbeTargetFormatsAreRhiPortable` (TestZFightingDetect.cpp) asserts every probe source's format is in the RHI-portable FLinearColor readback set and that depth keeps 4 bytes per channel; it fails on the old code because PF_R32_FLOAT is not in the set. Needs a live Vulkan run of `PinWright.render.capture_vocabulary+PinWright.render.detect_z_fighting` to verify.
