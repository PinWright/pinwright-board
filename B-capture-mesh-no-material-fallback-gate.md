---
id: B-capture-mesh-no-material-fallback-gate
title: "render.capture_mesh has no allowFallback material-fallback gate, so a mesh drawn with the default material returns success with no materialReadiness verdict"
status: OPEN
severity: High
category: bug
tags: [render, capture_mesh, material-fallback, materialReadiness, silent-false-success]
encounters: 1
lastSeen: 2026-10-03
rice: [1, 3, 1, 2]
priority: 90
---

# `render.capture_mesh` lacks the material-fallback gate its sibling capture verbs have

**Verified in source:** `B-capture-verbs-silent-default-material-fallback` `#2` gave `asset.generate_thumbnail` and `render.capture_asset_preview` the `allowFallback` param and the `materialReadiness` block (`PinWright::MaterialShaderState::ProbeCaptureAsset` / `ProbeCaptureComponent`, `AddCaptureReadiness`, `ApplyCaptureFallbackPolicy` in `Handlers/Material/MaterialShaderState.h`). `render.capture_mesh` (`Handlers/Render/RenderHandler.cpp`, `FMeshCaptureSession`) declares no `allowFallback` and publishes no `materialReadiness`. A slot whose shader map failed, or an unassigned drawn slot, renders as the engine default material and still returns `success:true`. The `readiness` block that `B-capture-mesh-cold-first-frame-no-readiness-wait` `#2` added reports shader-map completeness only. It does not say failed vs. compiling vs. default material.

**Fix:** after the final draw, probe the session's mesh component with `ProbeCaptureComponent`, publish `materialReadiness`, and apply `ApplyCaptureFallbackPolicy` behind a declared `allowFallback` param. That refuses with `MATERIAL_FALLBACK` by default, the same as `render.capture_asset_preview`. Add a test with a broken-material mesh fixture.

## History

- `#1-split-from-cold-frame-ticket` `OPEN` developer — Split out of `B-capture-mesh-cold-first-frame-no-readiness-wait`, whose Fix section asked for this gate. That ticket's `#2` did the readiness wait and settle redraw only. Severity High to match `B-capture-verbs-silent-default-material-fallback`.
