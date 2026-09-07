---
id: E-viewport-info-no-render-resolution
title: "No verb anywhere reports the resolution the renderer actually rasterises at: `system.inspect.get_viewport_info` returns `FViewport::GetSizeXY()` with no screen-percentage term, and the plugin has zero references to `GetRenderTargetSize` / `TemporalReprojection` / any render-resolution read — so with the editor window coming back a different size on every restart (measured 1515x939, then 738x502, then 1024x726, the small one reading ~50% faster) every before/after performance comparison PinWright's own `performance.*` verbs invite is unfalsifiable"
status: OPEN
severity: Medium
category: ergonomic
tags: [system, inspect, get_viewport_info, viewport, resolution, screen-percentage, tsr, upsampling, performance, profiling, benchmark, readback, missing-field, reproducibility, render, capture]
encounters: 1
costly: 1
lastSeen: 2026-08-29T20:10:00+03:00
---

# The one number every performance comparison depends on is the one number nothing reports

`system.inspect.get_viewport_info` reports the viewport's **logical** size. The renderer rasterises at
that size multiplied by the effective screen percentage — 83.9% on this host under TSR, so a 1511x939
viewport renders 1269x789, 42% fewer pixels. Nothing in the RPC surface reports the second number, or
the multiplier, or that a multiplier exists.

That would be a small omission if the first number were stable. It is not: the editor window does not
come back the same size across restarts, and the difference is large enough to swamp any change under
measurement.

## Mechanism, re-derived at HEAD `6d0e91a3`

    Handlers/Environment/EnvironmentHandler.cpp:993     REGISTER_RPC_HANDLER("system.inspect.get_viewport_info", ...)
    Handlers/Environment/EnvironmentHandler.cpp:994     RPC_NO_PARAMS
    Handlers/Environment/EnvironmentHandler.cpp:999-1001
        FViewport* Viewport = GEditor->GetActiveViewport();
        Resp->SetNumberField(TEXT("width"), Viewport->GetSizeXY().X);
        Resp->SetNumberField(TEXT("height"), Viewport->GetSizeXY().Y);

The remaining fields are camera-only — `cameraLocation` (`:1006`), `cameraRotation` (`:1008`), `fov`
(`:1010`), `success` (`:1012`) — which are `E-viewport-info-camera-transform`'s landed fix. There is
no screen-percentage, TSR or render-target term in the handler and no `r.ScreenPercentage` read.

**Swept the whole plugin, not just this handler:**

- `GetRenderTargetSize` — **zero** occurrences.
- `TemporalReprojection` — **zero** occurrences.
- `renderResolution` — **zero** occurrences.
- `ScreenPercentage` occurs only on **write** or request-echo paths:
  - `Handlers/Debug/PerformanceHandler.cpp:470-474` — `performance.set_resolution_scale` finds
    `r.ScreenPercentage` (`:470`), sets it (`:472`), and answers
    `Ctx.SendSuccess(TEXT("Resolution scale set"));` (`:474`) — a bare string with no read-back at all.
  - `Handlers/Environment/PostProcessHandler.cpp:392-399` — `post_process.set_anti_aliasing` writes
    `r.ScreenPercentage` (`:395`) and echoes `screenPercentage` (`:399`), which is the **requested
    input**, not a CVar read and not a resolution.

So the plugin can *set* the multiplier twice over and cannot *read* either the multiplier or the
result. Every `width`/`height` producer under `Handlers/` is a logical viewport size, a requested
capture size, an image dimension, or a window client size.

`sg.ResolutionQuality` compounds it: it is a scalability group that scales screen percentage
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Scalability.cpp:551` writes it at
`ECVF_SetByScalability`), so `performance.set_scalability` changes the render resolution as a side
effect and its readback (`PerformanceHandler.cpp:432-445`) reports the group's integer, not the pixels.

## Measured

Host `EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`; full method in
`Docs/map/vegetation-performance.md` § *Measurement hazards*, scripts under `dev/perf/`.

- **The window does not restore to the same size.** Across two editor restarts the level viewport came
  back **1515x939**, then **738x502**, then **1024x726**. `get_viewport_info` reports each of these
  truthfully and there is nothing else to compare them against.
- **A 738x502 viewport reads about 50% faster** on GPU frame time than a 1511x939 one. A change
  measured across a restart therefore arrives with a spectacular apparent win attached to it, and every
  other field in the comparison — draw calls, `RHI/PrimitivesDrawn`, `GPUSceneInstanceCount` — agrees
  with itself, so nothing in the data set flags it.
- **The real render size had to be read out of band.** 1511x939 logical vs **1269x789** actual, taken
  off `ProfileGPU`'s `TemporalReprojection(WxH)` line in `Saved/Logs/EAContentExamples58.log`. That is
  the only route that exists: run a profiling console command, then grep the editor log.
- **The class of error is not hypothetical on this project.** A whole baseline was discarded in the
  same session for the sibling reason — a stray `UnrealEditor-Cmd` process made every GPU pass ~40%
  more expensive at *identical resolution and identical draw calls* (P3 ShadowDepths 7.12 ms against
  3.90 ms). That one at least had a cause the agent could find; a resolution change hides inside the
  number the tool reports as correct.

**The workaround, which works and should not be necessary:** pin the window before baselining —
`editor.set_window_state {state: "restored"}` (`Handlers/Editor/EditorWindowHandlers.cpp:772`, params
`:778-783`) then `editor.resize_window {width: 2081, height: 1163}` (`:643`, params `:648-658`, which
sizes the **client area** and refuses a maximised window with `WINDOW_MAXIMIZED` at `:701-708`), which
yields 1511x939 on this machine. Then verify with `get_viewport_info` **and** with `ProfileGPU`,
because the first cannot see the screen percentage.

## Ask

1. **`renderWidth` / `renderHeight` and `screenPercentage` on `system.inspect.get_viewport_info`.** The
   handler already holds the `FViewport*` (`EnvironmentHandler.cpp:999`); the effective percentage is a
   `r.ScreenPercentage` read plus the viewport client's screen-percentage interface. Keep `width` /
   `height` exactly as they are — they are correct and callers depend on them — and add the derived
   pair beside them so the two can never be confused.
2. **A read-back on `performance.set_resolution_scale`** (`PerformanceHandler.cpp:470-474`), which
   currently answers a bare string. It writes the CVar that decides this; it should report the value
   that stuck and the resulting pixel dimensions. That also makes it the natural pinning verb.
3. **Include the render resolution in the profiling receipts** — whatever `performance.run_benchmark`
   and the CSV/`ProfileGPU` capture verbs return. A performance number without the resolution it was
   taken at is not comparable to anything, and these verbs exist to be compared across calls.

## Related

- **`E-viewport-info-camera-transform`** (DONE, Low) — the previous field added to this same verb, for
  the same reason: a capture could not be reasoned about without it. This is the next field along, and
  the argument is identical in shape.
- **`B-set-viewport-resolution-noop`** (IN-REVIEW, High) — `misc.set_viewport_resolution` echoes its
  input and never calls `r.SetRes`. That is the *write* side of the same blind spot; this is the read
  side. A caller today can neither set the viewport resolution through that verb nor observe what it
  actually is.
- **`B-performance-run-benchmark-no-completion-signal`**, **`E-performance-run-benchmark-async-poll-undocumented`**
  — the same verb family, and the same underlying point: its results are only meaningful against a
  pinned, reported rig.
- **`B-ortho-capture-culls-distant-foliage`**, **`B-capture-open-level-pose-params-photograph-stale-grass`**
  — capture-side siblings where the frame does not depict what the caller asked for. Same class,
  different axis.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive`, `B-mrq-render-result-omits-bitrate-and-size`: the
call succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported. `B-mrq-render-result-omits-bitrate-and-size` is the closest sibling — a render
receipt that omits the parameter that decides whether two outputs are comparable — and this is the same
defect one layer up, on the measurement rig rather than on the output file.

## Severity

**Medium.** Impact class is the rubric's Medium band verbatim: *a readback omits a field and forces a
fallback*. The fallback is real and was actually used — issue `ProfileGPU` through
`system.console_command`, then read `Saved/Logs/*.log` for the `TemporalReprojection(WxH)` line — which
is a log dive per measurement, on a verb family (`performance.*`) whose entire purpose is producing
comparable numbers.

**Not High.** No verb returns a false field. `width` and `height` are exactly what
`FViewport::GetSizeXY()` says and the verb never claims they are the rasterised size; the summary makes
no resolution promise it breaks. What makes it worse than a plain missing convenience — and what keeps
it out of Low — is that the omitted number **silently varies between sessions by a factor that dwarfs
what is being measured**, so its absence converts a correct readback into a wrong conclusion. That is
the same reasoning `E-python-get-editor-property-returns-live-view` used to argue a documentation-shaped
defect above Low.

**Not Low**, therefore, even though the fix is additive: Low is *pure friction ... a response spill that
only forces a `Read`*. This forces a different tool, a different file, and a judgement about whether the
last comparison was valid.

**Reach modifier declined.** `system.inspect.get_viewport_info` is a common verb — it is the documented
read-back companion to `editor.set_camera` and appears in most capture workflows, which argues for the
bump up. Declined because the *failure* only bites callers comparing two measurements taken at
different times, not the far larger population using the verb to confirm a camera pose; the reach of the
method is much wider than the reach of the defect. Medium stands.

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
