---
id: B-capture-render-resolution-unreported
title: "Every capture verb hands its requested width/height to the engine's editor screen-percentage heuristic and reports only the file size back: the frame is rasterised below W*H and TSR-upscaled to it, the fraction is computed FROM the requested size (so asking for a bigger image can lower the internal resolution), preview and level captures take two different resolution laws because GetViewStatusForScreenPercentage reads bIsRealtime and not IsRealtime(), a wireframe viewMode silently renders at 100% while the lit frame beside it does not, and the plugin has zero references to SetScreenPercentageInterface, GetRenderTargetSize or renderResolution anywhere"
status: OPEN
severity: High
category: bug
tags: [render, capture, camera, orbit_shots, frame_actor, capture_open_level, capture_asset_preview, screenshot, screen-percentage, resolution, upscaling, tsr, temporal-aa, anti-aliasing, missing-field, readback, silent, comparability, view-family, reported-not-reproduced]
encounters: 1
lastSeen: 2026-09-02T19:30:00+03:00
---

# `width: 1120, height: 1400` is the PNG. It is not what was drawn.

`CaptureEditorViewportToPng` (`PreviewViewportCaptureUtils.cpp:1643`) is the single primitive under
**every** capture verb in the plugin — `editor.screenshot`
(`Handlers/Editor/ViewportHandler.cpp:150`), the `render.capture_*` family
(`RenderHandler.cpp:1707`), `render.capture_annotated` (`AnnotatedCaptureHandler.cpp:541`), and every
pose-set verb (`camera.orbit_shots`, `camera.frame_actor`, `camera.animation_shots`,
`render.capture_animation_preview`) through `PoseListCapture.cpp:380`. It sizes the viewport to the
requested pixels, draws, reads back, and publishes the requested numbers:

    OutCapture.Width  = Request.Width;                              // PreviewViewportCaptureUtils.cpp:1666-1667
    SceneViewport->SetFixedViewportSize(Request.Width, Request.Height);   // :1768
    ColorData.Num() != Request.Width * Request.Height  -> capture fails   // :2203
    Result->SetNumberField(TEXT("width"), Capture.Width);           // RenderHandler.cpp:88, CameraShotPlanUtils.h:340

Those fields are **true about the file** — the readback refuses any buffer that is not exactly
W*H (`:2203`), so the PNG really is the size requested. They say nothing about the resolution the
scene was rasterised at, and no other field does either, because the plugin never sets one and never
reads one back.

## Reported observation — RELAYED from the rendering agent, not reproduced here

Read-only probes only during this audit; no capture was taken. Marked as relayed throughout.

`camera.orbit_shots` on a `staticMesh` subject, 1120x1400 requested, returned `width` 1120 /
`height` 1400 and PNGs of that size; a fluted column carried per-frame aliasing that moved shot to
shot across 240 stills, with `warmup.settleRounds 8` and `meanLuminanceDelta 2.7e-3`. Re-running the
set after issuing `r.ScreenPercentage 100` **and** `r.AntiAliasingMethod 0` gave `settleRounds 1`,
`meanLuminanceDelta 1.1e-6`, and visibly clean edges.

## Two claims in that report do not survive a source read, and the correction makes it worse

**1. `r.ScreenPercentage` is not the number in play.** Editor viewports ignore it, by design and by
comment:

    // In editor viewport, we ignore r.ScreenPercentage and FPostProcessSettings::ScreenPercentage by design.
    ViewFamily.SetScreenPercentageInterface(new FLegacyScreenPercentageDriver(
        ViewFamily, GlobalResolutionFraction));
    -- C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorViewportClient.cpp:4961-4963

I verified the cvar myself through `system.console.search` on this editor: `r.ScreenPercentage`
**71**, flags `["Scalability","Preview"]` — a scalability-preset write (the same one
`B-console-member-cvar-pin-freezes-scalability` records as sitting at `LastSetBy: Console`), and
irrelevant to an editor-viewport capture. `r.AntiAliasingMethod` **4** (TSR) is confirmed. So the
A/B above changed two cvars and only one of them can have acted; the improvement is **not**
attributable to the screen-percentage write.

**2. The wiki page named in the report,
`render.a-capture-response-is-sized-against-a-display-budget.md`, is about the 10,000-character
HTTP response spill threshold**, not about resolution. It does not warn about this and never has.
There is no page that does.

The correction is not a downgrade of the defect. `r.ScreenPercentage` is at least a number a caller
can read and set. What actually decides the capture's internal resolution is a heuristic run over
the capture's **own requested size**, and there is no cvar whose value is the answer.

## Mechanism, re-derived at plugin HEAD `47307435`, UE 5.8

### 1. The plugin installs no screen-percentage interface, anywhere

Swept `Plugins/PinWright/Source/` (`.cpp` + `.h`), whole tree, counts:

    SetScreenPercentageInterface              0
    LegacyScreenPercentageDriver              0
    PreviewResolutionFraction                 0
    SetPreviewScreenPercentage                0
    GetDefaultPrimaryResolutionFractionTarget 0
    GetRenderTargetSize                       0
    renderWidth / renderResolution            0
    TemporalReprojection                      0

The only `ScreenPercentage` hits in the plugin are `performance.set_resolution_scale` writing
`r.ScreenPercentage` (`Handlers/Debug/PerformanceHandler.cpp:470-474`) and
`post_process.set_anti_aliasing` echoing the requested `screenPercentage`
(`Handlers/Environment/PostProcessHandler.cpp:395-399`) — both of which write the cvar the editor
viewport ignores. This is the same sweep `E-viewport-info-no-render-resolution` ran on
`get_viewport_info`; it holds at this HEAD and it holds on the capture path too.

### 2. So the engine's editor fallback decides

`SceneViewport->Draw()` (`PreviewViewportCaptureUtils.cpp:117`, inside `PumpViewport` `:107-125`)
lands in `FEditorViewportClient::Draw`, where:

    if (ViewFamily.GetScreenPercentageInterface() == nullptr)          // EditorViewportClient.cpp:4940
    {
        float GlobalResolutionFraction = 1.0f;
        if ((!bStereoRendering || bInVREditViewMode) &&
            SupportsPreviewResolutionFraction() && ViewFamily.SupportsScreenPercentage())   // :4945-4946
        {
            ... GlobalResolutionFraction = GetDefaultPrimaryResolutionFractionTarget();      // :4956
            ViewFamily.EngineShowFlags.ScreenPercentage = (GlobalResolutionFraction != 1.0); // :4958
        }
        ViewFamily.SetScreenPercentageInterface(new FLegacyScreenPercentageDriver(...));     // :4962
    }

and the target is a static heuristic seeded with the **viewport's own pixel count**:

    FStaticResolutionFractionHeuristic StaticHeuristic;
    StaticHeuristic.Settings.PullEditorRenderingSettings(GetViewStatusForScreenPercentage());  // :3540
    StaticHeuristic.TotalDisplayedPixelCount = FMath::Max(Viewport->GetSizeXY().X * Viewport->GetSizeXY().Y, 1);  // :3546
    StaticHeuristic.DPIScale = GetDPIScale();                                                   // :3547
    return StaticHeuristic.ResolveResolutionFraction();                                         // :3548

`Viewport->GetSizeXY()` at that moment **is the capture's requested size** — `SetFixedViewportSize`
resizes the viewport synchronously and the plugin's own comment says so
(`PreviewViewportCaptureUtils.cpp:1766-1768`). The resolution fraction is therefore a function of
`width`/`height`, which is the parameter the caller uses to ask for more detail.

### 3. Which branch a capture takes is decided by a member the plugin's realtime override does not touch

    else if (!bIsRealtime) { return EViewStatusForScreenPercentage::NonRealtime; }   // EditorViewportClient.cpp:3523

`bIsRealtime` is the raw member (`EditorViewportClient.h:2169`), written only by `SetRealtime`
(`EditorViewportClient.cpp:780`). `AddRealtimeOverride` — which is what the capture pushes at
`PreviewViewportCaptureUtils.cpp:1726-1727`, deliberately, because an asset-editor preview viewport
is not realtime (`:1718-1725`) — only appends to `RealtimeOverrides` (`:713-718`), and that array is
consulted by `IsRealtime()` (`EditorViewportClient.h:414-417`) and **not** by
`GetViewStatusForScreenPercentage`. So:

- **asset-preview captures** (`camera.orbit_shots` on a mesh/animation subject,
  `render.capture_asset_preview`) -> `NonRealtime` -> mode from
  `r.Editor.Viewport.ScreenPercentageMode.NonRealTime`;
- **level captures** (`editor.screenshot`, `render.capture_open_level`, `camera.frame_actor` on the
  level viewport) -> `Desktop` -> mode from `r.Editor.Viewport.ScreenPercentageMode.RealTime`.

Two capture families, two different resolution laws, from one verb surface that documents neither.
Read off this editor with `system.console.search` (mine, verified):
`ScreenPercentageMode.NonRealTime` **2** (`BasedOnDPIScale`), `ScreenPercentageMode.RealTime` **1**
(`BasedOnDisplayResolution`), `r.Editor.Viewport.ScreenPercentage` 100 (Manual-mode only, so unused
here), `MinRenderingResolution` 720, `MaxRenderingResolution` 2160.
Enum order: `Manual=0, BasedOnDisplayResolution=1, BasedOnDPIScale=2`
(`Runtime/Engine/Public/LegacyScreenPercentageDriver.h:66-76`).

### 4. What each law works out to

`ResolveResolutionFraction` (`Runtime/Engine/Private/LegacyScreenPercentageDriver.cpp:352-447`),
with `GetRenderingPixelCount(h) = h*h*(1920/1080)` (`:103-106`) and the anchors from
`Engine/Config/BaseEngine.ini:2248-2254` (`Min 720/720, Mid 2160/1080, Max 4320/1440`):

- **`BasedOnDPIScale`** (`:418-421`): `fraction = 1.0f / DPIScale`. At 100% Windows scaling that is
  1.0 and there is no upscale at all; at 125% it is 0.80; at 150%, 0.67. The internal resolution of
  an asset-preview capture is therefore set by the **operating system's display scaling**, and two
  hosts running the same script produce different pictures with identical responses.
- **`BasedOnDisplayResolution`** (`:361-414`) for the reported 1120x1400 = 1,568,000 px:
  `t = (1568000 - 921600) / (8294400 - 921600) = 0.0877`;
  `lerped = lerp(921600, 2073600, 0.0877) = 1,022,600`;
  `fraction = sqrt(1022600/1568000) = ` **0.808**, i.e. rasterise **904x1131** and TSR-upscale to
  1120x1400 — 65% of the delivered pixels. Neither clamp binds (`Max` gives 2.30 at `:428-435`,
  `Min` gives 0.767 at `:437-444`).
  Because the lerp is anchored at a *display* pixel count, the fraction **falls as the request
  grows**: 2240x2800 = 6,272,000 px gives `t = 0.726`, `lerped = 1,757,500`,
  `fraction = sqrt(1757500/6272000) = ` **0.529** — rasterise 1186x1482 for a 2240x2800 file.
  Doubling each edge to get more detail quadruples the delivered pixels and only 1.7x's the drawn
  ones. Asking for a bigger image lowers the internal resolution.

### 5. `viewMode` silently moves it again

`SupportsPreviewResolutionFraction()` returns false for `VMI_Wireframe`, `VMI_BrushWireframe`,
`VMI_LightComplexity`, `VMI_LightmapDensity`, `VMI_ReflectionOverride`, `VMI_LODColoration`,
`VMI_CollisionPawn`, `VMI_ShadowCasters` and the rest of that list
(`EditorViewportClient.cpp:3474-3500`), and the whole `:4945-4959` block is gated on it — so those
captures render at fraction 1.0. A caller who pairs a lit frame with a diagnostic frame at the same
requested size through PinWright's own `viewMode` parameter gets two **different internal
resolutions**, reported identically. That pairing is a documented workflow of these verbs.

### 6. Temporal AA is left on, and two sibling capture paths in this same plugin deliberately turn it off

Nothing on the `CaptureEditorViewportToPng` path touches `TemporalAA` or the AA method. The plugin's
other two capture paths both suppress it, and both record why:

    Component->ShowFlags.SetTemporalAA(false);
    // ... the renderer would otherwise jitter the projection per frame and blend against history,
    // so two tiles of one mosaic would differ by their own sub-pixel offsets at the seam.
    // Off is what makes a burst comparable with itself.
    -- Handlers/Render/OrthoTileCaptureUtils.cpp:503 (comment :500-502; base flags :172)

    // Without a persistent view state the capture has no temporal history, which forces
    // FSceneView::SetupAntiAliasingMethod down to AAM_None and leaves no TAA/TSR jitter. That is
    // exactly what this probe wants: two captures of the same world state differ ONLY by the
    // parameter the caller perturbed, with no temporal noise underneath.
    -- Handlers/Render/SceneCaptureProbeUtils.cpp:49-54

The argument those two comments make is *precisely* the argument for an orbit set: 240 stills of one
subject, each meant to differ only by camera azimuth. The viewport path is the one that does not
make it, and it is the path every capture verb runs on. `r.AntiAliasingMethod 4` on this editor, so
TSR is what those frames went through, and each shot begins with a camera jump that invalidates the
reprojection history the method needs.

## What I did NOT establish

- **No capture was taken.** Every pixel-level observation above is the rendering agent's; the
  fractions in section 4 are arithmetic from engine source, not measured output.
- **`DPIScale` on this host is unread**, so I cannot say which of 1.0 / 0.8 / 0.67 the reported
  asset-preview run actually used, only that it came from that law and not from `r.ScreenPercentage`.
- **The A/B is confounded** (two cvars at once) and the screen-percentage half is inert on this path
  by `:4961`, so the only supported reading of "clean edges at `r.AntiAliasingMethod 0`" is that
  **AA**, not screen percentage, produced the visible improvement. Whether the capture was also
  upscaled at the time is unverified — it depends on the DPI scale, and on that branch it may well
  have been 1.0.
- The route that would settle it without guessing is the one
  `E-viewport-info-no-render-resolution` had to use: `ProfileGPU` through
  `system.console_command`, then grep `Saved/Logs/*.log` for `TemporalReprojection(WxH)`. That it is
  still the only route is this ticket.

## Ask

1. **Publish what was drawn.** `viewport.render: {width, height, resolutionFraction, antiAliasingMethod, upscaled}`
   beside the existing `width`/`height`, which stay exactly as they are (they are correct about the
   file and callers depend on them). Everything needed is on hand at the capture site: the client is
   already held, `GetDefaultPrimaryResolutionFractionTarget()` is public
   (`EditorViewportClient.h:1651` neighbourhood, impl `:3536-3548`), and `r.AntiAliasingMethod` is a
   cvar read. This is the minimum that makes an upscaled capture distinguishable from a native one,
   and it is additive.
2. **Pin the capture instead of inheriting the editor's.** The seam exists and is already used for
   exactly this class of problem — `FScopedCaptureProjectionAspect`
   (`PreviewViewportCaptureUtils.cpp:1712-1713`), the scoped view-mode override, the exposure pin,
   `FScopedViewDistanceScale` (`:161-162`). Add a scoped
   `ViewportClient.SetPreviewScreenPercentage(100) + SetPreviewingScreenPercentage(true)`
   (`EditorViewportClient.cpp:3557-3587`) for the capture's duration, so a verb that promises W*H
   draws W*H. Default it on; a `screenPercentage` parameter can expose the old behaviour for anyone
   who wants the cheap frame.
3. **Suppress temporal AA for a still, or report that it was not suppressed.** Same decision the
   ortho-tile and scene-capture-probe paths already made and documented
   (`OrthoTileCaptureUtils.cpp:503`, `SceneCaptureProbeUtils.cpp:49-54`). A capture set whose whole
   purpose is that its members differ only by the requested parameter cannot leave a per-frame jitter
   phase in the frames.
4. **Say it on the docs page.** There is currently no wiki text anywhere warning that a capture is
   not rendered at the size it returns — the page the report thought carried this warning
   (`render.a-capture-response-is-sized-against-a-display-budget`) is about response byte size.

## Related

- **`E-viewport-info-no-render-resolution`** (OPEN, Medium) — the same blind spot on the
  *measurement* rig: `system.inspect.get_viewport_info` reports the logical viewport with no
  screen-percentage term. That ticket's plugin-wide sweep is the one re-run above and it still holds.
  **The two are not duplicates**: that one is a missing field on a read-only inspection verb whose
  fix is additive; this one is a capture verb that produces a file at a size it did not draw at, and
  its fix (Ask 2) changes what the pixels are. Its § *Ask 1* and this § *Ask 1* should be
  implemented together — same derived numbers, two response shapes.
- **`B-set-viewport-resolution-noop`** (IN-REVIEW, High) — `misc.set_viewport_resolution` echoes its
  input and never calls `r.SetRes`. The write side of the same blind spot.
- **`B-preview-rig-first-capture-stale-sky`** (OPEN, Medium) — same capture call, same session, same
  numbers. Its § *Ask 3* (warn when `settled` is true at the `MaxWarmupSettleRounds` ceiling) is the
  settle half of this observation; see its `#2` for why that signature turns out to track the
  editor's AA method rather than pending preview-scene work.
- **`B-exposure-pin-black-frame`** (IN-REVIEW, High) — the ticket that built the settle loop, and the
  precedent for pinning a render setting per capture rather than inheriting the editor's.
- **`B-showflag-cvar-override-contaminates-capture`**, **`B-game-view-shared-state-no-capture-warning`**
  — the standing class: a capture inherits editor-global render state and does not say so. This is
  that class on the one piece of state that decides how many pixels were drawn.
- **`B-performance-typed-verbs-pin-scalability-cvars`** (IN-REVIEW, High) — records
  `r.ScreenPercentage` among the scalability cvars PinWright writes; worth reading before anyone
  "fixes" this by setting that cvar, which an editor viewport ignores.

## Same shape as

`B-mrq-render-result-omits-bitrate-and-size` — a render receipt that omits the parameter deciding
whether two outputs are comparable — except that here the omitted parameter is derived from the
caller's own input, so the receipt is not merely incomplete: the one knob the caller has for "more
detail" moves it in the wrong direction, silently.

## Severity

**High.** Impact class alone is the rubric's **Medium** band verbatim — *a readback omits a field and
forces a fallback* — and the fallback is real and documented (`ProfileGPU` + log grep, per
`E-viewport-info-no-render-resolution`). The **reach modifier carries it up one**: the affected code
is the single primitive under every capture verb in the plugin
(`ViewportHandler.cpp:150`, `RenderHandler.cpp:1707`, `AnnotatedCaptureHandler.cpp:541`,
`PoseListCapture.cpp:380`), captures run in nearly every session on this project, and the failing
population is not a corner of the callers — it is all of them, on every host whose DPI scale is not
1.0 or whose captures go through the level viewport.

**Not Critical.** Nothing crashes and no asset data is written or lost.

**High not Medium, on a second independent ground:** the defect is not only an omission. The
resolution fraction is computed *from the caller's requested size*
(`EditorViewportClient.cpp:3546`), so the parameter the caller reaches for to get more detail can
reduce the internal resolution; and `viewMode` moves it again (`:3474-3500`, `:4946`). A caller
comparing two captures — the primary use of `camera.orbit_shots` and of every opposed-view workflow
these verbs document — is comparing frames drawn at different resolutions with nothing in either
response to say so. That is closer to *silent wrong conclusions on a normal path* than to a missing
convenience field.

**Held at High rather than argued higher because no field returned is false.** `width` and `height`
are exactly the PNG's dimensions and the readback enforces it (`:2203`); the verbs make no claim
about the render resolution that they then break. The same reasoning that kept
`E-viewport-info-no-render-resolution` at Medium applies here, and the difference in the verdict is
entirely the reach modifier plus the request-dependent fraction.

## History
- `#1-capture-inherits-the-editor-screen-percentage` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase render. Pixel observations RELAYED from the rendering agent and NOT reproduced here (read-only probes only, no captures): `camera.orbit_shots` staticMesh subject at 1120x1400 returned `width` 1120 / `height` 1400 and PNGs of that size with shot-to-shot edge aliasing across 240 stills, `warmup.settleRounds 8`, `meanLuminanceDelta 2.7e-3`; a re-run after `r.ScreenPercentage 100` + `r.AntiAliasingMethod 0` gave `settleRounds 1`, `1.1e-6` and clean edges. TWO REPORTED PREMISES ARE WRONG and are corrected in the body: (a) editor viewports ignore `r.ScreenPercentage` by design (`EditorViewportClient.cpp:4961-4963`), so that half of the A/B is inert and the improvement is attributable to the AA cvar only — the A/B changed two variables at once; (b) the wiki page the report cited, `render.a-capture-response-is-sized-against-a-display-budget`, is about the 10,000-character response spill threshold, not resolution, and no page warns about this. Mechanism re-derived at plugin HEAD `47307435`, UE 5.8: `CaptureEditorViewportToPng` (`PreviewViewportCaptureUtils.cpp:1643`) is the one primitive under every capture verb (`ViewportHandler.cpp:150`, `RenderHandler.cpp:1707`, `AnnotatedCaptureHandler.cpp:541`, `PoseListCapture.cpp:380`); it sets the viewport size to the request (`:1768`), publishes the request back (`:1666-1667`, `RenderHandler.cpp:88-89`, `CameraShotPlanUtils.h:340-341`) and enforces it on readback (`:2203`), and never installs a screen-percentage interface — plugin-wide sweep counts ZERO for `SetScreenPercentageInterface`, `LegacyScreenPercentageDriver`, `PreviewResolutionFraction`, `SetPreviewScreenPercentage`, `GetDefaultPrimaryResolutionFractionTarget`, `GetRenderTargetSize`, `renderWidth`, `renderResolution`, `TemporalReprojection`. So the engine fallback decides (`EditorViewportClient.cpp:4940-4964`) via `GetDefaultPrimaryResolutionFractionTarget` (`:3536-3548`), which seeds the heuristic with `Viewport->GetSizeXY()` (`:3546`) — the capture's OWN requested size — and the DPI scale (`:3547`). Branch selection reads the raw `bIsRealtime` member (`:3523`, decl `EditorViewportClient.h:2169`, written only by `SetRealtime` `:780`), which the capture's `AddRealtimeOverride` (`PreviewViewportCaptureUtils.cpp:1726-1727`; engine `:713-718`) does not touch because only `IsRealtime()` consults the override array (`EditorViewportClient.h:414-417`) — so asset-preview captures take the NonRealtime law and level captures the RealTime law. Cvars READ BY ME on this editor via `system.console.search`: `r.ScreenPercentage` 71 flags [Scalability, Preview] (scalability preset, ignored here), `r.AntiAliasingMethod` 4 (TSR), `r.Editor.Viewport.ScreenPercentageMode.NonRealTime` 2 = BasedOnDPIScale, `.RealTime` 1 = BasedOnDisplayResolution, `r.Editor.Viewport.ScreenPercentage` 100 (Manual-only), Min/MaxRenderingResolution 720/2160. Arithmetic from `LegacyScreenPercentageDriver.cpp:352-447` + `BaseEngine.ini:2248-2254`: NonRealtime gives `1/DPIScale`; RealTime at 1120x1400 gives fraction 0.808 (rasterise 904x1131, 65% of delivered pixels) and FALLS as the request grows (~0.62 at 2240x2800). `viewMode` moves it a third time — `SupportsPreviewResolutionFraction()` is false for wireframe/light-complexity/LOD-coloration modes (`:3474-3500`) so those render at 1.0 while the lit frame beside them does not. Temporal AA is never suppressed on this path, while the plugin's two OTHER capture paths suppress it deliberately and give this exact argument for doing so (`OrthoTileCaptureUtils.cpp:503` with comment `:500-502`, `SceneCaptureProbeUtils.cpp:49-54`). NOT ESTABLISHED: no capture taken; `DPIScale` on this host unread, so which NonRealtime fraction the reported run used is unknown; the fractions above are arithmetic, not measured. Ask: publish `viewport.render.{width,height,resolutionFraction,antiAliasingMethod,upscaled}`; pin the capture with a scoped `SetPreviewScreenPercentage(100)` (`EditorViewportClient.cpp:3557-3587`) alongside the existing scoped-pin seams; suppress temporal AA for a still or report that it was not; and say it in the docs. Severity High: impact class is Medium (`readback omits a field and forces a fallback`, the fallback being ProfileGPU + log grep), bumped by the reach modifier because the primitive is under every capture verb and the failing population is all callers, and argued independently on the request-dependent fraction turning `width` into a control that moves detail the wrong way. Not Critical (no crash, no data loss); not Medium because of reach plus that second ground; no returned field is false, which is why it is not argued higher still."

- `#2-corrected-the-large-request-fraction` `OPEN` reporter — "Correction to `#1`, made in the same session and before any reader acted on it. `#1` states the `BasedOnDisplayResolution` fraction at 2240x2800 as `~0.62`; the arithmetic is wrong and the body now carries the right figure. Redone from `LegacyScreenPercentageDriver.cpp:361-414` with the `BaseEngine.ini:2248-2254` anchors: displayed 6,272,000 px, `t = (6272000 - 921600) / (8294400 - 921600) = 0.726`, `lerped = lerp(921600, 2073600, 0.726) = 1,757,500`, `fraction = sqrt(1757500/6272000) = 0.529`, i.e. rasterise 1186x1482 for a 2240x2800 file; neither clamp binds (`Max` 1.150 at `:428-435`, `Min` 0.383 at `:437-444`). The direction of `#1`'s claim is unaffected and gets stronger, not weaker: doubling each requested edge quadruples the delivered pixels while raising the drawn ones only 1.7x. The 1120x1400 figure in `#1` (fraction 0.808, 904x1131) was re-checked and stands."
