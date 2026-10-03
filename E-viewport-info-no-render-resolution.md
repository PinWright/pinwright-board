---
id: E-viewport-info-no-render-resolution
title: "system.inspect.get_viewport_info reports only the logical viewport size and performance.set_resolution_scale returns a bare string, so the live viewport's render resolution (logical size x effective screen percentage) cannot be read through any verb"
status: OPEN
severity: Medium
category: ergonomic
tags: [system, inspect, get_viewport_info, viewport, resolution, screen-percentage, tsr, upsampling, performance, profiling, benchmark, readback, missing-field, reproducibility, render, capture]
encounters: 1
costly: 1
lastSeen: 2026-08-29T20:10:00+03:00
rice: [1, 2, 1, 2]
priority: 8
---

# No verb reads the live viewport's render resolution

`system.inspect.get_viewport_info`
(`Source/PinWright/Private/Handlers/Environment/EnvironmentHandler.cpp:994`, `RPC_NO_PARAMS`
`:995`) returns `width`/`height` from `FViewport::GetSizeXY()` (`:1001-1002`), plus `pie`,
`activeViewport` and the editor camera. The renderer rasterises that size times the effective
screen percentage (83.9% under TSR on the reporting host: a 1511x939 viewport rendered 1269x789).
Nothing reports the second number or the fraction.

`performance.set_resolution_scale` (`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp:466-487`)
writes `r.ScreenPercentage` at its existing priority (`:482-483`) and answers
`SendSuccess(TEXT("Resolution scale set"))` (`:485`) with no read-back.

Capture verbs no longer have this gap: they publish `viewport.render {width, height,
screenPercentage, ...}` (`Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp:3668-3684`,
tracked by `B-capture-render-resolution-unreported`). That block is not a substitute here, because a
capture pins the fraction to 1.0 for its own frame (`CaptureResolutionViewExtension.cpp:30-35`), so it
describes the capture, not what the interactive viewport renders during a benchmark. Reading the
`r.ScreenPercentage` CVar is not enough either: the editor viewport's effective fraction also comes
from the editor screen-percentage heuristic.

Why it matters: the editor window came back 1515x939, then 738x502, then 1024x726 across restarts on
the reporting host, and the small one read ~50% faster on GPU frame time with every other reported
field self-consistent. Performance comparisons across sessions are not comparable without the render
size, which today only `ProfileGPU`'s `TemporalReprojection(WxH)` line in the editor log provides.

**Workaround:** pin the window before baselining (`editor.set_window_state {state:"restored"}`, then
`editor.resize_window` to a fixed client size), then confirm the render size with `ProfileGPU` via
`editor.console_command` and read the log.

**Fix:** on `system.inspect.get_viewport_info`, add `renderWidth`, `renderHeight` and
`screenPercentage` measured from the level viewport's last view family (the same primary x secondary
fraction `CaptureResolutionViewExtension` reads), keeping `width`/`height` unchanged. On
`performance.set_resolution_scale`, return the re-read `r.ScreenPercentage` value and its SetBy
priority, the way `set_scalability` and `apply_baseline_settings` re-read theirs.

**Acceptance:** with the viewport at 1511x939 and TSR's default fraction, `get_viewport_info` reports
`width:1511, height:939` and render dimensions matching `ProfileGPU`'s `TemporalReprojection(WxH)`;
`performance.set_resolution_scale {scale:50}` returns the re-read `r.ScreenPercentage` (50, or the
clamped value) instead of a bare string.

## Related

- `B-capture-render-resolution-unreported` (IN-REVIEW) — the capture half, already reported.
- `F-performance-frame-time-statistics` — wants the same render size on the `run_benchmark` receipt.
- `B-set-viewport-resolution-noop` (IN-REVIEW) — the write side of viewport resolution.
- `E-perf-wp-configure-readback-thin` — the bare-string pattern across the `performance.*` CVar
  setters, of which `set_resolution_scale` is one.

## History
- `#1-logical-viewport-is-not-the-render-size` `OPEN` reporter — Filed from a performance-profiling pass
  on `/Game/Maps/PW_VegetationTest` (host `EAContentExamples58`, UE 5.8); method and numbers in
  `Docs/map/vegetation-performance.md`, scripts under `dev/perf/`. Verified at HEAD `6d0e91a3`:
  `system.inspect.get_viewport_info` (`Handlers/Environment/EnvironmentHandler.cpp:993`, `RPC_NO_PARAMS`
  `:994`, body `:995-1021`) builds `width`/`height` from `Viewport->GetSizeXY()` at `:999-1001` with no
  screen-percentage term; remaining fields are the camera trio from `E-viewport-info-camera-transform`
  (`:1006`, `:1008`, `:1010`). Plugin-wide sweep: **zero** occurrences of `GetRenderTargetSize`,
  `TemporalReprojection` or `renderResolution`, and every `ScreenPercentage` hit is a write or a
  request echo — `performance.set_resolution_scale` sets `r.ScreenPercentage`
  (`PerformanceHandler.cpp:470-472`) and answers a bare `SendSuccess(TEXT("Resolution scale set"))`
  (`:474`) with no read-back; `post_process.set_anti_aliasing` echoes the requested `screenPercentage`
  (`PostProcessHandler.cpp:399`), not a read. `sg.ResolutionQuality` compounds it — it scales screen
  percentage (`Scalability.cpp:551`) so `performance.set_scalability` moves the render resolution and
  reports the group integer instead. Measured: the editor viewport came back **1515x939, then 738x502,
  then 1024x726** across two restarts, the small one reading **~50% faster** on GPU frame time with
  every other reported field self-consistent; the true render size (1269x789 against a 1511x939 logical
  viewport, 83.9% TSR input) was obtainable only from `ProfileGPU`'s `TemporalReprojection(WxH)` line in
  the editor log. Same session lost an entire baseline to the sibling effect (a second engine process
  made every GPU pass ~40% dearer at identical resolution and draw calls, ShadowDepths 7.12 vs 3.90 ms).
  Working pin, recorded because the ask does not remove the need for it:
  `editor.set_window_state {state:"restored"}` (`EditorWindowHandlers.cpp:772`) then
  `editor.resize_window {width:2081, height:1163}` (`:643`, client-area sizing, `WINDOW_MAXIMIZED` gate
  `:701-708`). Ask: `renderWidth`/`renderHeight`/`screenPercentage` beside the existing `width`/`height`
  (not replacing them), a read-back on `performance.set_resolution_scale`, and the render resolution in
  the profiling receipts. Severity Medium on the readback-omits-a-field band; High declined because no
  field returned is false and the verb makes no resolution promise; Low declined because the omitted
  number varies silently between sessions by more than the effect under measurement; reach bump declined
  because the defect's population (callers comparing two measurements) is much smaller than the verb's.
- `#2-rephrased` `OPEN` developer — Old text claimed no verb anywhere reports render resolution and asked for it on every profiling receipt; capture verbs now publish `viewport.render {width, height, screenPercentage}` (`PreviewViewportCaptureUtils.cpp:3668-3684`, `B-capture-render-resolution-unreported`), though pinned to 100% for the capture (`CaptureResolutionViewExtension.cpp:30-35`) so it does not describe the live viewport. Narrowed to `get_viewport_info` render size plus a `set_resolution_scale` read-back (now `PerformanceHandler.cpp:466-487`, bare string `:485`), moved the benchmark-receipt ask to `F-performance-frame-time-statistics`, shortened the title, cut the severity essay and the plugin-wide sweep, refreshed citations (`EnvironmentHandler.cpp:994`, `:1001-1002`), added Acceptance. Severity unchanged (Medium: readback omits a field and forces a log-dive fallback). RICE effort E 1->2: the live viewport's effective fraction has to be measured off its view family (a view-extension read, as captures do), not a CVar read.
