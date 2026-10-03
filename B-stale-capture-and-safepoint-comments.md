---
id: B-stale-capture-and-safepoint-comments
title: "Two stale source comments: CaptureReadinessGate.h says NullRHI has no shader compiling manager (it always has one), SafePoint.cpp cites the thumbnail render at the wrong line"
status: OPEN
severity: Low
category: bug
tags: [comments, docs, capture, readiness-gate, safepoint, nullrhi, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
rice: [1, 1, 1, 1]
priority: 8
---

# Stale comments in CaptureReadinessGate.h and SafePoint.cpp

1. **`Utils/CaptureReadinessGate.h:42-44`** documents `FPendingWork::bShaderCompilerAvailable` as
   "False on a host with no shader compiling manager (a NullRHI or commandlet run)". The engine
   creates `GShaderCompilingManager` on both branches of `AllowShaderCompiling()`
   (`C:\UE_5.8\Engine\Source\Runtime\Launch\Private\LaunchEngineLoop.cpp:3245-3259`; the else branch
   comments "create a manager, but it won't do anything internally"), and
   `Utils/CaptureReadinessGate.cpp:37-39` sets the flag whenever the pointer is non-null. Under
   NullRHI the published `shaderCompilerAvailable` (`CaptureReadinessGate.cpp:94`, documented in
   `docs/wiki-src/render.md:46`) is therefore `true`, and the shader count is reported as a
   measurement. A reader of the header expects `false` there.
2. **`Dispatch/SafePoint.cpp:108-109`** justifies listing `asset.generate_thumbnail` with
   "ThumbnailTools::RenderThumbnail - an offscreen scene render on the calling stack
   (AssetWorkflowHandler.cpp:716)". Line 716 of `Handlers/Asset/AssetWorkflowHandler.cpp` is now
   asset-rename code; the `ThumbnailTools::RenderThumbnail` call is at `:1389` (inside
   `RenderThumbnailPass`, `:1386`).

**Impact:** documentation only; a reader verifying the claims is sent the wrong way. No behaviour
change.
**Fix:** (1) reword the header comment to what the code does (false only when
`GShaderCompilingManager` is null, which the editor never observes; under NullRHI the manager exists
and simply has no work), or drop the flag if nothing can make it false. (2) Cite the function
(`RenderThumbnailPass` in `AssetWorkflowHandler.cpp`) instead of a line number.

## History
- `#1-two-stale-comments` `OPEN` reporter — Found in today's verification; both verified against source (engine `LaunchEngineLoop.cpp:3245-3259`, plugin `AssetWorkflowHandler.cpp:1389`).
