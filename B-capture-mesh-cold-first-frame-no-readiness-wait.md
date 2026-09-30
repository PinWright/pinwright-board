---
id: B-capture-mesh-cold-first-frame-no-readiness-wait
title: "render.capture_mesh renders one CaptureScene with no material/texture/Nanite readiness wait and no cold-frame retry, so the first capture after editor boot returns an unsettled frame with success:true and a hard-coded redrawRetries:0"
status: OPEN
severity: High
category: bug
tags: [render, capture_mesh, cold-frame, warmup, readiness, streaming, shader-map, silent-false-success, visual-verification, first-frame]
encounters: 1
lastSeen: 2026-09-30
---

# The first `render.capture_mesh` after boot is cold and the response cannot say so

**Observed (relayed, not reproduced here):** two identical calls (propeller mesh, `location z16`, `pitch -90`, `exposure 0`) right after editor boot. First: `sizeBytes 542845`, `litPixelFraction 0.928`, `flatRegionFraction 0.0688`, `redrawRetries 0`, mesh missing/unsettled. Second: `sizeBytes 563570`, `litPixelFraction 0.99997`, `flatRegionFraction 0.731`, pixel-identical to the reference.

**Verified in source (PinWright submodule):**
- The only readiness step is mesh compilation: `FStaticMeshCompilingManager::FinishCompilation` / `FSkinnedAssetCompilingManager::FinishCompilation` (`Handlers/Render/MeshPreviewCaptureUtils.cpp:201,214`). Nothing waits on the mesh's materials' game-thread shader maps, texture streaming (`WaitForStreaming` / `IStreamingManager::StreamAllResources`), or Nanite page streaming.
- Each shot is a single synchronous `CaptureScene()` with no view state (`Handlers/Render/SceneCaptureProbeUtils.cpp:338,384`), read back once (`MeshPreviewCaptureUtils.cpp:326`); no settle loop, no degenerate-frame check, no retry.
- `redrawRetries` is published from `Capture.RedrawRetries` (`Handlers/Render/RenderHandler.cpp:186`), which defaults to 0 (`PreviewViewportCaptureUtils.h:979`) and is only ever set by the editor-viewport path (`PreviewViewportCaptureUtils.cpp:2784`). On this verb the field is a constant, not a measurement; `bWarmupMeasured = false` (`MeshPreviewCaptureUtils.cpp:84`) is not surfaced as a warning.
- `capture_mesh` does not take the `allowFallback` material-readiness gate that `B-capture-verbs-silent-default-material-fallback` `#2` added to `generate_thumbnail` / `capture_asset_preview` (params at `RenderHandler.cpp:345-368`).

Mechanism for this specific frame (shader map vs. texture mips vs. Nanite pages vs. sky capture) is not traced; the missing wait covers all candidates. The sky-capture drain is already reported via `captureIncomplete` (`B-preview-rig-first-capture-stale-sky` `#6`) and is not the only gap.

**Workaround:** take and discard one warm-up `render.capture_mesh` call on the same asset before any capture you trust; compare `flatRegionFraction` / `litPixelFraction` across the pair.

**Fix:** reuse `WaitForThumbnailSubjectReadiness` (`Handlers/Asset/ThumbnailFrameEvidence.h:91`, shipped by `B-thumbnail-cold-first-frame-no-stats` `#2`) on the mesh plus its used materials/textures in `FMeshCaptureSession::Create`; wait for Nanite streaming residency where the mesh is Nanite; publish the readiness report; add one measurement-gated re-render (compare two consecutive `CaptureColor` frames, retry once if they differ) and set `redrawRetries` from it, or omit the field on this verb. Extend the material-fallback gate to `capture_mesh`.

## History

- `#1-filed-from-source` `OPEN` reporter — Filed from a subagent's relayed pair of identical post-boot `render.capture_mesh` calls (numbers above). Verified from source only (no MCP calls; editor busy): mesh-compile wait is the only readiness step, one `CaptureScene` per shot, `redrawRetries` is structurally 0 on this path. Dedup: `B-thumbnail-cold-first-frame-no-stats` (DONE, `asset.generate_thumbnail` only — its readiness helper was never wired into `capture_mesh`, so not a regression), `B-preview-rig-first-capture-stale-sky` (IN-REVIEW, sky-capture drain, already reported by `captureIncomplete`), `B-capture-verbs-silent-default-material-fallback` (IN-REVIEW, permanent material failure, does not cover this verb), `F-mesh-capture-without-asset-editor-or-world-lock` (IN-REVIEW, introduced the verb; this is a defect in it, not a return of the feature). Severity High to match the sibling cold-frame ticket: silent wrong evidence on the only mesh-author capture route.
