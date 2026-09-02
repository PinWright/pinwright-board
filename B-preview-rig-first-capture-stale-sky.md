---
id: B-preview-rig-first-capture-stale-sky
title: "No preview-scene capture ever runs an editor tick, so the sky-capture update never runs inside a capture call — the rig is reported applied while the pixels are lit by whatever the last editor frame left, and the settle loop certifies it because its pump cannot drive the work it is waiting for"
status: OPEN
severity: Medium
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, previewScene, preview-scene, rig, sky-light, skylight-capture, warmup, settle, stale, silent-wrong-data, first-frame, reported-not-reproduced]
encounters: 3
lastSeen: 2026-09-02T20:50:00+03:00
---

# `applied: true` and `settled: true` over pixels the rig never reached

**Reported by the capture agent on 2026-09-02 and NOT reproduced by me** — no capture was taken
during this audit (read-only probes only). The mechanism below **is** mine, re-derived from source;
the observation is relayed and is marked as such throughout.

## Reported observation

    camera.orbit_shots {subject:{kind:"staticMesh", path:"/Game/Atlantis/Meshes/SM_Column_Doric",
                                 closeAfterCapture:false},
                        radius:5600, fov:34, width:1120, height:1400,
                        previewScene:{showEnvironment:true, showFloor:true},
                        exposure:{mode:"auto"},
                        angles:[{azimuth:0,elevation:8},{azimuth:90,elevation:8}]}

the first call after flipping the profile's stored `showFloor` from false to true. Response:
`viewport.previewScene.applied: true`, `showFloor: true`, `showEnvironment: true`, key and sky
intensity 1, `warmup: {settled: true, settleRounds: 8, meanLuminanceDelta: 0.0027}`. The az-0
pixels were a near-black silhouette on a near-black floor (`subjectRegion.subject.meanLuminance
0.607`, `unlitFraction 0.033`, `minLuminance 0`, `subjectRegionWarning` fired). The identical pose
re-shot one call later, identical rig, same pinned gain (`adapted 4.880198` both times), was
brightly lit.

## Mechanism: a capture call never runs the tick that updates sky captures

**1. The preview scene's update runs on the editor tick, not on Slate's.**

    class FAdvancedPreviewScene : public FPreviewScene, public FTickableEditorObject
    -- C:/UE_5.8/Engine/Source/Editor/AdvancedPreviewScene/Public/AdvancedPreviewScene.h:30

with `virtual void Tick(float DeltaTime) override` declared under its `FTickableEditorObject`
block (`:45-49`). `FTickableEditorObject`s are driven by the editor frame loop, not by
`FSlateApplication::Tick`.

**2. That tick is the only thing that updates sky captures for a preview world.**
`FAdvancedPreviewScene::Tick` (`AdvancedPreviewScene.cpp:279`) opens with

    UpdateCaptureContents();
    -- AdvancedPreviewScene.cpp:282

and closes with the recapture block

    if (bSkyChanged)
    {
        SkyLight->SetCaptureIsDirty();
        SkyLight->MarkRenderStateDirty();
        SkyLight->UpdateSkyCaptureContents(PreviewWorld);
        ...
    }
    -- AdvancedPreviewScene.cpp:314-323

`FPreviewScene::UpdateCaptureContents` is `USkyLightComponent::UpdateSkyCaptureContents(PreviewWorld)`
plus the reflection-capture equivalent, and its own comment names its only three callers —
*"This function is called from FAdvancedPreviewScene::Tick, FBlueprintEditor::Tick, and
FThumbnailPreviewScene::Tick, so assume we are inside a Tick function"* (`PreviewScene.cpp:245-253`).

This matters because `SetCaptureIsDirty()` does not recapture anything — it queues the component:

    SkyCapturesToUpdate.AddUnique(this);
    -- Runtime/Engine/Private/Components/SkyLightComponent.cpp:364 (in SetCaptureIsDirty, :354-368)

and `UpdateSkyCaptureContents` is what drains that queue. A dirty sky light stays stale until a
tick drains it.

**3. A PinWright capture never runs an editor frame.** The whole per-round pump is:

    void PumpViewport(FEditorViewportClient& ViewportClient, const TSharedPtr<FSceneViewport>& SceneViewport)
    {
        FSlateApplication& SlateApp = FSlateApplication::Get();
        SlateApp.PumpMessages();
        SlateApp.Tick(ESlateTickType::All);
        ViewportClient.Invalidate();
        if (SceneViewport.IsValid()) { SceneViewport->Invalidate(); SceneViewport->Draw(); }
        if (FSlateRenderer* Renderer = SlateApp.GetRenderer()) { Renderer->FlushCommands(); }
        FlushRenderingCommands();
    }
    -- Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.cpp:107-125

Slate messages, a viewport invalidate, a draw, two flushes. No editor frame, so no
`FTickableEditorObject::Tick`, so no `FAdvancedPreviewScene::Tick`, so **no sky-capture update at
any point inside a capture call** — not before the first shot, not between shots, not during the
settle loop. Whatever the sky light's captured state was when the last real editor frame ended is
what every shot in the call is lit by.

**4. And the rig never asks for one either — which is stronger than "the recapture is deferred".**
The rig's two writes that bear on sky lighting both take paths that leave `bSkyChanged` false:

- floor / environment go through the `bDirect=true` branches, whose entire bodies are
  `FloorMeshComponent->SetVisibility(...)` (`AdvancedPreviewScene.cpp:405-407`) and
  `SkyComponent->SetVisibility(...)` (`:422-426`). Neither touches `SkyLight` or `bSkyChanged`.
- sky intensity goes through `FPreviewScene::SetSkyBrightness`, whose whole body is
  `SkyLight->SetIntensity(SkyBrightness)` (`PreviewScene.cpp:318-324`).

`bSkyChanged` is set at exactly three sites, all inside `FAdvancedPreviewScene::UpdateScene`
(`:148`, `:167`, `:191`) — reachable only via `UAssetViewerSettings::PostEditChangeProperty`, which
is the broadcast `FScopedPreviewSceneRig` **deliberately avoids**, on record and for a good reason:

    //  * SetFloorVisibility(bVisible) with bDirect defaulted false calls PostEditChangeProperty on
    //    the process-wide UAssetViewerSettings (AdvancedPreviewScene.cpp:391-403), which
    //    broadcasts to every live preview scene in the editor. That IS Defect 1.
    -- Source/PinWright/Private/Handlers/Render/PreviewSceneRig.cpp:687-689

So the guard's own (correct) fix for the shared-profile leak is what removes the only signal that
would have marked the sky capture dirty. The rig then calls `Client.Invalidate()` and sets
`bApplied = true` (`PreviewSceneRig.cpp:736-737`) — an invalidate schedules a **redraw**, which is
not a tick and updates no captures. `applied: true` is therefore a true statement about the writes
and not a statement about the pixels, and nothing in the response distinguishes the two.

## Why `warmup.settled` cannot catch this

The settle loop pumps and re-reads until the frame's mean luminance stops moving
(`PreviewViewportCaptureUtils.cpp:2245-2270`). Its pump is the `PumpViewport` above — so **the loop
cannot advance the work it would need to detect**. A frame lit by a stale sky capture is not a
frame in transition; it is a finished frame of the wrong thing, and it converges immediately and
legitimately. A convergence test whose pump cannot drive the pending work always reports
convergence. The comment at `:2226-2244` is careful that `settled` is a measurement and not a
promise, and it is — the measurement is just blind to this class.

The reported numbers contain the one available tell and it is never surfaced.
`MaxWarmupSettleRounds = 8` (`PreviewViewportCaptureUtils.h:323`), and the loop increments before
testing, so `settleRounds: 8` with `settled: true` means the frame was still moving through the
entire budget and stopped on the last round permitted — against a healthy call's 1–2. The response
publishes `settleRounds` (`:3244`) but the warning is gated on the negative case only:

    if (!Capture.bWarmupSettled) { ... "warmupWarning" ... }
    -- PreviewViewportCaptureUtils.cpp:3247-3260

so "settled on round 1" and "barely settled at the budget ceiling" are reported identically except
for one number nobody is told to read. The tolerances make the gap concrete: settled pairs were
measured at 1.5e-5 and 1.1e-4 against an absolute tolerance of 1e-3
(`PreviewViewportCaptureUtils.h:306-319`), while the reported `meanLuminanceDelta` was **0.0027** —
25x the worst measured settled pair and *above* the absolute tolerance, passing only on the
relative term.

## What I did NOT establish, and how to close it

The reporter's hypothesis is that the sky light's **captured irradiance** still reflects the
previous rig. The deferral half is confirmed and is worse than reported (never requested at all,
per 4 above). The **carrier** is not confirmed, and one engine fact argues against the specific
link to `showFloor`: `FAdvancedPreviewScene` installs the sky light from the profile's cubemap
asset — `SetSkyCubemap(Profile.EnvironmentCubeMap.Get())` (`AdvancedPreviewScene.cpp:57`), and
`USkyLightComponent::SetCubemap` sets `Cubemap` + `MarkRenderStateDirty` + `SetCaptureIsDirty`
(`SkyLightComponent.cpp:1018-1029`) — so on `SLS_SpecifiedCubemap` the irradiance derives from that
asset, not from the scene, and toggling the floor mesh's visibility cannot change what it would
capture. Two other candidates fit the same "one tick behind" shape and are not excluded:

- a pending capture queued by something else and drained only on the next tick, since
  `UpdateSkyCaptureContentsArray` additionally **defers** while textures, meshes or shaders are
  async-compiling and re-tries an incomplete capture no sooner than 5 s later
  (`SkyLightComponent.cpp:737-741`, `:757-787`) — which would make the first capture after any
  asset load stale for reasons the response also never mentions;
- the newly-visible floor's own render state landing a frame after the draw that read the pixels.

Whoever fixes this should instrument rather than assume: log `SkyCapturesToUpdate.Num()` and the
component's `CaptureStatus` immediately before the first `ReadPixels` of a rig-changing call, and
compare against the second call. **The ticket does not depend on which candidate wins** — the
defect established here is the invariant, not the carrier: no capture call runs the tick that
completes preview-scene work, and both `applied` and `settled` are reported as though it had.

## Ask

In descending order of value, and none of them requires a reproduction to justify:

1. **Run the tick, or say that it was not run.** Drive one `FAdvancedPreviewScene::Tick` (or the
   `FPreviewScene::UpdateCaptureContents()` it opens with) after the rig is applied and before the
   first real shot — the rig already owns the "apply once for the whole set" seam
   (`CameraFrameHandler.cpp:706`) and the primitive already takes one discarded warm-up frame per
   call for a neighbouring reason (`PoseListCapture.h:158-169`), so there is a natural place for it.
   If driving an editor tick from a handler is judged unsafe, publish
   `previewScene.captureUpdated: false` so `applied` stops implying it.
2. **Split `applied`.** `applied: true` currently means "the writes were made". A caller reading it
   next to a picture reads "these pixels are that rig". Report the two separately.
3. **Warn at the budget ceiling.** Emit a `warmupWarning` (or a distinct `settleWarning`) when
   `bWarmupSettled` is true but `WarmupSettleRounds == MaxWarmupSettleRounds`, or when
   `meanLuminanceDelta` passes only on the relative term. Today the whole signal is one unremarked
   integer; the shape of the existing `hideWarning` / `restoreWarning` pairs is the precedent.

## Related

- `B-capture-asset-preview-renders-foliage-black` (High, IN-REVIEW) — **the ticket a triager is
  most likely to confuse this with**, and the distinction should be checked before either is
  worked: that one is foliage-only, reproducible on every call, and has an established cause; this
  is a static mesh, transient, and clears on the very next call with the same rig and the same
  pinned gain. Same reported symptom vocabulary, different defect.
- `B-exposure-pin-black-frame` (High, IN-REVIEW) — the warm-up settle loop and its `warmupWarning`
  exist because of that ticket's 2.26-stop first frame. This is the same failure shape one layer
  down: the detector built there is blind to work its own pump cannot drive.
- `B-capture-open-level-pose-params-photograph-stale-grass` (High, DONE) — the structural twin on
  the level side: a capture photographs derived state built for a different configuration while
  every reported field is correct.
- `B-capture-preview-decoration-not-suppressible` (Medium, OPEN) — the other open defect in the
  `previewScene` rig's surface; its `#2` records what `showFloor` / `showEnvironment` do and do not
  reach.
- `B-capture-render-resolution-unreported` (High, OPEN) — filed from the same session and the same
  frames. It carries the anti-aliasing half of this observation (the capture path never suppresses
  temporal AA, while the plugin's two other capture paths deliberately do), and its evidence is why
  `#2` below reclassifies the `settleRounds: 8` signature this ticket's `#1` read as a tell.

## Severity

**Medium.** Impact class is High by the rubric — silent wrong data on a normal path, and the caller
is actively told otherwise twice (`applied: true`, `settled: true`) — but the reach modifier takes
it down one: the trigger is the first capture after a `previewScene` change on an asset-preview
verb, not an every-session path, and the frame is visibly wrong rather than subtly so, so it fails
loudly to a human looking at the picture. Held at Medium also because the observation is
**reported and not reproduced here**; a reproduction showing it survive into a plausible-looking
frame would argue for High.

## History
- `#1-no-tick-inside-a-capture` `OPEN` reporter — "Filed 2026-09-02 from the Atlantis showcase video. Observation RELAYED from the capture agent, not reproduced during this audit (read-only probes only): first `camera.orbit_shots` after flipping the profile's stored `showFloor` false->true returned `previewScene.applied:true` + `warmup:{settled:true, settleRounds:8, meanLuminanceDelta:0.0027}` over a near-black frame; identical pose one call later, same rig, same `adapted 4.880198`, brightly lit. Mechanism re-derived from source: `FAdvancedPreviewScene` is an `FTickableEditorObject` (`AdvancedPreviewScene.h:30,45-49`) whose `Tick` (`:279`) opens with `UpdateCaptureContents()` (`:282`) and closes with the `bSkyChanged` recapture (`:314-323`); `FPreviewScene::UpdateCaptureContents` is the only sky-capture drain on this path and documents its three tick-only callers (`PreviewScene.cpp:245-253`); `SetCaptureIsDirty` only queues (`SkyLightComponent.cpp:354-368`). PinWright's `PumpViewport` (`PreviewViewportCaptureUtils.cpp:107-125`) pumps Slate and draws, never running an editor frame, so no capture call ever updates sky captures. The rig also never sets `bSkyChanged`: its floor/env writes use the `bDirect=true` branches (`AdvancedPreviewScene.cpp:405-407,422-426`) and its sky write is `SkyLight->SetIntensity` only (`PreviewScene.cpp:318-324`), because the `PostEditChangeProperty` path that sets `bSkyChanged` (`:148,167,191`) is deliberately avoided as Defect 1 (`PreviewSceneRig.cpp:687-689`). The settle loop's pump is that same `PumpViewport`, so it cannot drive the pending work and converges on a stable wrong frame; `settleRounds:8` equals `MaxWarmupSettleRounds` (`PreviewViewportCaptureUtils.h:323`) and the warning is gated on `!bWarmupSettled` only (`:3247-3260`), so the one tell is never surfaced. NOT established: that the carrier is specifically the sky light's captured irradiance — the preview sky light is a specified cubemap (`AdvancedPreviewScene.cpp:57`, `SkyLightComponent.cpp:1018-1029`), so `showFloor` cannot change what it would capture; two other one-tick-behind candidates are listed in the body with an instrumentation plan. The invariant, not the carrier, is what this ticket asserts."

- `#2-settle-ceiling-signature-tracks-the-aa-method` `OPEN` reporter — "Additional evidence, same session (Atlantis showcase render, 2026-09-02), RELAYED from the rendering agent and again NOT reproduced here (read-only probes only, no captures). New trigger, and it is not a rig change: a 240-still `camera.orbit_shots` set on a `staticMesh` subject at 1120x1400 with NO `previewScene` write returned `warmup: {settled: true, settleRounds: 8, meanLuminanceDelta 2.7e-3}` on EVERY still — the same signature `#1` recorded as the one available tell that a frame was still moving through the whole budget. The same poses re-shot after setting `r.AntiAliasingMethod 0` returned `settleRounds 1`, `meanLuminanceDelta 1.1e-6`. I verified `r.AntiAliasingMethod` = 4 (TSR) on this editor myself via `system.console.search`. THE A/B IS CONFOUNDED — the run also set `r.ScreenPercentage 100` — but editor viewports ignore `r.ScreenPercentage` by design (`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorViewportClient.cpp:4961-4963`), so the AA cvar is the only one of the two that can have acted. WHAT THIS CHANGES HERE, and it is an interpretation rather than a mechanism: `#1`'s arithmetic is right — the loop condition is `WarmupSettleRounds < MaxWarmupSettleRounds` (`PreviewViewportCaptureUtils.cpp:2249`) and the counter increments (`:2259`) before the tolerance test (`:2266-2272`), so `settled: true` with `settleRounds: 8` really does mean the frame converged on the last permitted round and a genuinely unconverged run reports `settled: false` — but that signature is NOT SPECIFIC to pending preview-scene work. On any host running a temporal AA method the residual is dominated by TSR's per-frame jitter phase, and 2.7e-3 clears the gate only on the relative term (`WarmupSettleRelTolerance = 0.005`, `PreviewViewportCaptureUtils.h:319`, so it needs a frame mean above ~0.54; the absolute term is 1e-3 at `:318`). The metric is also blind in the direction that matters: the test is on the FRAME MEAN (`:2261-2268`) while the visible defect in those 240 stills was per-pixel edge jitter, which barely moves a mean — so `settled` was simultaneously a true statement about the measurement and a false implication about the frames. CONSEQUENCE FOR THIS TICKET'S ASK 3: the ceiling warning is still the right ask and this makes it more valuable, but implemented as written it will fire on essentially every capture on every TSR host, so it must name the anti-aliasing method alongside the round count rather than attributing the residual to warm-up. THE ANTI-ALIASING HALF IS FILED SEPARATELY as `B-capture-render-resolution-unreported` (High, OPEN) — the capture path never suppresses temporal AA while the plugin's two other capture paths deliberately do and give this exact argument for it (`Handlers/Render/OrthoTileCaptureUtils.cpp:503` with comment `:500-502`, `Handlers/Render/SceneCaptureProbeUtils.cpp:49-54`) — so it is not duplicated here. Status and severity UNCHANGED: this evidence bears on the reading of one number and on Ask 3's shape; it does not touch the invariant `#1` establishes (no capture call runs the editor tick that completes preview-scene work), and the new trigger produced no wrong picture of its own."
- `#3-frame-mean-blindness-measured-and-ask-1-seam-does-not-exist` `OPEN` reporter — Additional evidence plus one correction to this ticket's own Ask. Status and severity deliberately UNCHANGED; `encounters` 2 -> 3. **`#2`'s frame-mean point is now MEASURED rather than inferred, and on a cleaner signature than `#2`'s.** Same verb, same subject, same day (`camera.orbit_shots`, `/Game/Atlantis/Meshes/SM_Column_Doric`, `previewScene {showFloor:true, showEnvironment:true}`, pinned `ev100 -2.2869`, 1120x1400, 18 shots = 6 azimuths x 3 repeats of the identical pose in ONE call; response `Saved/PinWright/HttpResponses/20260902T092641Z/20260902T171435Z_388389c9-…json`). Where `#2` had the ceiling signature (`settleRounds:8`, `meanLuminanceDelta 2.7e-3`), this set reports the **cleanest possible** result — `settled:true, settleRounds:1, meanLuminanceDelta 2.0808e-5, settleMs 33.37` — while the delivered pixels, decoded off disk with a stdlib decoder (whole-frame means reproduce the response's own `imageStats.meanLuminance` to 4 dp), show a 28x28 patch on the subject's base plinth cycling **27.98/255 with period exactly 4 by capture index** (variance explained 1.000, linear R^2 0.000, phase sd 0.08-0.26). Arithmetic against the gate: the last shot's frame mean is 0.6898, so the tolerance is `max(1e-3, 0.005 * 0.6898) = 3.449e-3` (`PreviewViewportCaptureUtils.h:318-319`) and the reported delta cleared it by **166x**; the local swing is 0.1097 normalised, i.e. **32x the tolerance and ~5,270x the reported delta**; and the whole-frame mean moves only ~0.001 inside a repeat triplet, **0.29x the tolerance**. So the settle criterion is not merely blind to this class — it certifies it with two orders of magnitude to spare, and does so on the round-1 path where `#2`'s ceiling tell is absent by construction. **CONSEQUENCE FOR ASK 3:** the `settleRounds == MaxWarmupSettleRounds` warning would NOT have fired here. It remains worth having, but it is necessary and not sufficient; what is needed is a change of measurement, from a whole-frame scalar to a per-pixel one, and the helper already exists in this same primitive — `PinWrightPoseCapture::MeasureChangedPixelFraction` (`PoseListCapture.cpp:502-541`, declared `PoseListCapture.h:368`), already called per shot for the coverage differential (`:316`), with pixel retention already plumbed (`Frame.bRetainPixels`, `:297`). **LINE-CITATION REFINEMENT to `#2`**, which cited `:2261-2268` for "the test is on the FRAME MEAN": that range covers the delta expression (`:2261`) and the tolerance (`:2267-2268`) but not the loop bound (`:2249`), the exit (`:2269-2273`) or the time bound (`:2274-2277`); the whole block is `PreviewViewportCaptureUtils.cpp:2246-2280`. And the quantity itself is worth pinning: `ImageStats.MeanLuminance` is a **whole-frame, full-resolution, unweighted** Rec.709 luma mean over every pixel of the readback — `CalculateCaptureImageStats`, `:464-547`, per-pixel luma `:483-486`, accumulated `:487`, divided `:496`. It is not downsampled and not a histogram; the 256-bin histogram at `:480`/`:492` feeds only `toneLevelsUsed` and never the settle delta. **CORRECTION TO ASK 1, and it changes what that ask can be built on.** Ask 1 states that "the rig already owns the 'apply once for the whole set' seam (`CameraFrameHandler.cpp:706`)". That line is not a seam — it is the `previewScene` **parameter description** on `camera.orbit_shots`, i.e. a promise published to callers: *"Read once for the whole set, so every shot in one call is lit identically, and restored once after the last one."* The code does not do that. `FScopedPreviewSceneRig` is constructed at **exactly one site in the whole tree**, `PreviewViewportCaptureUtils.cpp:1906`, inside `CaptureEditorViewportToPng` — **per shot**, not per set. Two sibling comments assert the same false thing (`CameraFrameHandler.cpp:774-777`; `:1135-1136`, which spells out "so no two shots in one set can be lit differently from each other or from their own report"). It is provable from the response alone, with no source reading: `viewport.previewScene` is reported from the last capture and reads `showFloor: true` applied with `previous.showFloor: false` and `afterRestore.showFloor: false` — the floor was off entering the last shot of a set that requested it throughout, because the previous shot's destructor had put it back. So Ask 1's proposed home for the tick does not exist yet: hoisting the rig to the set has to happen first or alongside. **Filed separately as `B-preview-capture-lighting-cycles-per-shot` (OPEN/High)** — the steady-state complement to this ticket's first-capture transient: that cycle persists indefinitely instead of clearing on the next call, and its three tests (84-93% of significant differences off high-gradient edges, non-zero phase-coherent flat-region signed means, filled-face rather than rim-shaped difference images) rule out the TSR-jitter attribution `#2` reached for. This ticket's own invariant — that no capture call runs the editor tick which completes preview-scene work — is untouched and unretracted.
