---
id: B-capture-asset-preview-renders-foliage-black
title: "render.capture_asset_preview renders foliage subjects as a black silhouette inside a correctly lit preview environment, so the one verb for reviewing a plant before planting it cannot judge one — all four subjects render correctly in-level, and no field in the response distinguishes a black subject from a dark one"
status: OPEN
severity: High
category: bug
tags: [render, capture_asset_preview, foliage, vegetation, preview-scene, black-subject, lighting, shader-compile, no-readback, unknown-cause]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The environment lights. The plant does not.

Four foliage assets captured with `render.capture_asset_preview`:

- `HillTree_P2` — `blank: false`, mean luminance **0.033**, whole scene black.
- Three other foliage assets — a **correctly lit environment** with a **pure black subject**.

**All four render correctly in-level.** The assets and their materials are fine; the defect is in
the preview path.

## Not the alpha-zero false alarm

Stating this explicitly because it is the standing trap on this project, and a reviewer who skips
it will close this ticket for the wrong reason.

A blank-looking frame is very often alpha-0 with intact RGB. That is **not** what this is:

1. `blank: false` and a non-zero mean luminance mean the frame carries content. The classifier is
   `Handlers/Render/PreviewViewportCaptureUtils.cpp:520` —
   `Stats.bBlank = Stats.MeanLuminance <= BlankMeanLuminance && Stats.LitPixelCount < LitPixelFloor`
   — computed off the `ReadPixels` buffer (`:2099`), so it is measuring real returned colour.
2. Three of the four frames show a **correctly lit environment around a black subject**. An alpha
   problem is a property of the whole surface; it does not selectively black out one object and
   leave the backdrop, floor and gradient intact. A frame that is right everywhere except the
   subject is a shading or a lighting result, not an encode result.

## What the preview path actually is — verified in source

There is no PinWright-built preview scene to blame. **The verb opens the real asset editor and
reads back its live viewport.** `Handlers/Render/` contains no `FPreviewScene::ConstructionValues`,
no `UWorld::CreateWorld`, no `EWorldType::EditorPreview`, no thumbnail renderer:

- `Handlers/Render/CaptureSubject.cpp:1548-1549` — `FindEditorForAsset` then `OpenEditorForAsset`.
- `CaptureSubject.cpp:1570-1579` — finds the toolkit's `SEditorViewport`, takes its
  `GetViewportClient()` and `GetSceneViewport()`.
- `Handlers/Render/PreviewSceneRig.cpp:115` — `Client.GetPreviewScene()`, cast to
  `FAdvancedPreviewScene` at `:124`.
- Capture is a Slate viewport readback, not a scene capture component:
  `Handlers/Render/PreviewViewportCaptureUtils.cpp:1614`
  `OutCapture.Renderer = TEXT("sceneViewportReadPixels")`, pixels at `:2099`.

So the lighting is the engine's `FAdvancedPreviewScene` — a directional key **and** a sky light and
a sky sphere, all created by the engine, none created or destroyed here. `PreviewSceneRig.cpp:438-
453` only **reads** them (`KeyIntensity` `:443`, `SkyIntensity` `:448`, `bSkyVisible` `:449`,
`SkyCubemapPath` `:450-452`); the write path at `:696-734` runs only when the caller passes
`previewScene`, and every field of `FPreviewSceneRigPin` defaults to not-provided
(`PreviewSceneRig.h:76-102`). The verb forces exactly one preview default, `bShowFloor = false` for
Niagara subjects (`RenderHandler.cpp:918-923`).

Four candidate causes are therefore **ruled out in source, not guessed**:

- `ShowFlags.Lighting` is never written. It is read at `PreviewViewportCaptureUtils.cpp:349` and
  nowhere else; `SetLightingOnlyOverride` does not appear in `Handlers/Render/` at all.
- No view-mode override by default — `FScopedViewModeOverride` returns without writing unless an
  explicit `viewMode` was passed (`:1411-1416`).
- No skylight is removed or dimmed by default (see above).
- No Lumen/GI manipulation on this path.

## What IS verified missing, and why it is probably still not the cause here

**`render.capture_asset_preview` waits for no compilation.** `grep -rn` for
`FAssetCompilingManager`, `GShaderCompilingManager`, `FinishAllCompilation` and
`FinishCompilationForObjects` across all of `Handlers/Render/` returns **nothing**. The sibling
verb does the opposite: `asset.generate_thumbnail` blocks on
`FAssetCompilingManager::Get().FinishCompilationForObjects(WaitSet)`
(`Handlers/Asset/ThumbnailFrameEvidence.cpp:61`) before drawing, with the reasoning at `:57-60`.
The only synchronisation on the preview path is `FlushRenderingCommands()` inside `PumpViewport`
(`PreviewViewportCaptureUtils.cpp:123`) plus a warm-up **mean-luminance settle loop**
(`:2145-2172`, `bWarmupSettled` at `:2168`) — and a mean over the whole frame can converge while a
small subject is still on a fallback shader, because the backdrop dominates the mean.

That is a real defect and the closest mechanism on the board
(`B-thumbnail-cold-first-frame-no-stats`, DONE — a missing compile wait producing a wrong frame
under a healthy payload). **But the evidence here argues against it being this cause**, and the
ticket should say so rather than hand a fixer a lead that dies: the same four assets rendered
correctly in-level in the same editor session, which means their materials were already compiled.
A missing compile wait would also be expected to be flaky across four captures rather than
producing four consistent failures. Worth fixing on its own merits; do not expect it to close this
ticket.

## What is NOT established

The cause. Listed candidates, flagged honestly:

- **The preview profile's environment being off.** `FAdvancedPreviewScene`'s `bShowEnvironment`
  hides the sky sphere and starves the skylight, which would remove exactly the indirect and
  transmitted light a two-sided-foliage shading model depends on while a directional key still
  lights opaque surfaces correctly. That would produce this signature — black plant, lit
  backdrop — but **it was not checked.** This is the strongest untested candidate and it is cheap:
  the response already publishes the answer. `PreviewSceneRig.cpp:456-471` measures
  `bShowFloor` / `bShowEnvironment` / `bRotateLightingRig` and
  `PreviewViewportCaptureUtils.cpp:2799-2801` emits all three, alongside key and sky readings from
  `:438-453`. **A fixer's first move should be to re-run one capture and read
  `showEnvironment`, `skyIntensity` and `keyIntensity` off the response.** Those fields were
  present in the measured captures and were not recorded.
- **The fixed key-light direction versus a two-sided card.** A foliage card lit from behind on a
  shading model without a transmission term reads black. Not verifiable from plugin source — the
  key rotation belongs to the engine's advanced preview scene — and not measured.
- **Whether the preview world has Lumen / DFAO / sky occlusion at all.** An engine and
  `AssetViewerSettings` property, not touched by this plugin, not inspected.

No claim is made about which of these it is. `grep` over the whole `Handlers/Render/` tree for
`two-sided`, `twosided`, `subsurface`, `transmission` and `shading model` returns zero hits, so
nothing in the plugin is doing anything foliage-specific — which is itself the useful negative: if
the cause is in the preview scene's lighting state, the fix is to pin that state for the capture
rather than inherit whatever the user's editor was left in.

## What the 2026-08-29 pass established — three candidates REFUTED in engine source

No live editor was available (the MCP gateway refused with `EDITOR_NOT_RUNNING`, and 27 other
agents held the checkout mid-edit), so the one-call measurement this ticket asks for was not taken.
What *was* possible is reading the engine, and it kills three of the leads above outright. Engine
source is `C:\UE_5.8\Engine\Source` on this host (not the path the project CLAUDE.md names).

1. **`bShowEnvironment` CANNOT starve the skylight.** This was the ticket's strongest untested
   candidate and it is wrong as stated. `FPreviewScene`'s skylight is created with
   `SourceType = SLS_SpecifiedCubemap` (`Runtime/Engine/Private/PreviewScene.cpp:92`) and
   `FAdvancedPreviewScene` feeds it `Profile.EnvironmentCubeMap` through `SetSkyCubemap`
   (`Editor/AdvancedPreviewScene/Private/AdvancedPreviewScene.cpp:58`, `PreviewScene.cpp:326-332`).
   It reads a cubemap, never the scene. `bShowEnvironment` drives only the sky SPHERE mesh's
   visibility (`AdvancedPreviewScene.cpp:225`, `SkyComponent->SetVisibility`). Hiding the backdrop
   removes a backdrop; the indirect light keeps arriving.
2. **The skylight's own on/off switch is a DIFFERENT flag, and it is already reported.**
   `Profile.bUseSkyLighting` drives `SkyLight->SetVisibility` (`AdvancedPreviewScene.cpp:229`), and
   PinWright already reads exactly that: `PreviewSceneRig.cpp:449`,
   `Report.bSkyVisible = Scene->SkyLight->GetVisibleFlag()`, emitted as `sky.visible`
   (`PreviewViewportCaptureUtils.cpp`, `MakeOnePreviewSceneRigObject`). So the response has always
   carried the answer to "is the skylight on", separately from `showEnvironment`.
3. **The preview key light already has the transmission term.** `PreviewScene.cpp:87` sets
   `DirectionalLight->bTransmission = true` on the preview directional light. The "a two-sided
   foliage card lit from behind reads black for want of a transmission term" candidate is refuted
   for the preview key specifically.
4. **The compile-wait candidate predicts a GREY subject, not a black one.** When a material's
   shader map is incomplete the renderer walks the fallback proxy chain until it finds a complete
   one, terminating at the default material —
   `FMaterialRenderProxy::GetMaterialWithFallback`, `Runtime/Engine/Private/Materials/
   MaterialRenderProxy.cpp:865-888`, the `do { GetFallback(...) } while
   (!Material->IsRenderingThreadShaderMapComplete())` loop. The default material is grey. This is
   independent corroboration of the ticket's own argument against the compile wait, on firmer
   ground than "the assets were already compiled in-session".
5. **A new fact about the pump, for whoever does add the compile wait.** `PumpViewport`
   (`PreviewViewportCaptureUtils.cpp:106-125`) runs `FSlateApplication::Tick`,
   `SceneViewport->Draw()` and `FlushRenderingCommands()`. None of those drives
   `FAssetCompilingManager::ProcessAsyncTasks`, which is the game-thread step that FINALISES a
   finished shader map (the whole argument of `Utils/AssetCompilePump.h`). So the warm-up settle
   loop cannot advance an in-flight compile no matter how long it runs — it settles onto the
   fallback frame. A compile wait here must call `PinWright::AssetCompile::AdvanceOnGameThread()`
   or `FinishCompilationForObjects`; more viewport pumps would not have worked.

**Net effect on the candidate list:** the preview scene's lighting is a much weaker suspect than the
ticket assumed, and the compile wait is weaker still. Nothing here identifies the cause. It remains
unestablished, which is why this ticket stays OPEN.

## Consequence

The verb cannot be used to judge a foliage asset at all. That matters more than it sounds: this is
the natural way to review a plant *before* planting it, and vegetation is the asset category where
reviewing first matters most, because the alternative is discovering the problem after a scatter has
already placed several thousand instances of it.

The failure is also not self-announcing to a caller that does not look at the pixels. `blank: false`
is **correct** — the frame genuinely is not blank — and no field in the response describes the
subject separately from the frame. A caller that gates on `blank` or on mean luminance passes a
useless image through.

## Fix direction

Pin the preview scene's lighting for the capture instead of inheriting the editor's current
profile: the write path already exists (`PreviewSceneRig.cpp:696-734`) and is currently opt-in via
`previewScene`. Making a known-good sky and environment the default for this verb — or at minimum
warning when `showEnvironment` is false, the way `rotateLightingRig` already warns
(`PreviewViewportCaptureUtils.cpp:2916`) — turns an inherited editor state into a reported one.
Establish the cause first; the measurement named above costs one call.

Separately and independently: add a compile wait to this path, matching
`ThumbnailFrameEvidence.cpp:61`. It is a verified gap regardless of whether it is this bug.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. `blank`, `meanLuminance` and `litPixelCount` are all correct
measurements *of the frame*; none of them is a measurement of the subject, which is the thing the
verb exists to show.

Nearest members, and why none of them is this:

- `B-capture-preview-default-camera-vs-light` (OPEN, Medium) — **the closest ticket, and it
  explicitly disclaims this case.** Its body corrects the "dark side" story, argues the default
  pose is front-lit, and says the luminance gap it measured is a frame mean dominated by the
  backdrop, "not evidence the subject is better shaded". It names the discriminator it could not
  supply; this finding is that discriminator — a subject-specific failure with the environment
  provably lit. Cross-link, do not fold.
- `B-capture-asset-preview-renders-empty` (IN-REVIEW, High) — the opposite signature: the *whole*
  frame solid white with a gizmo, caused by a non-realtime viewport, and already fixed by the
  `AddRealtimeOverride` at `PreviewViewportCaptureUtils.cpp:1673`. A lit environment around a black
  subject is not that failure.
- `B-exposure-pin-black-frame` (IN-REVIEW, High) — an all-black *frame* from exposure crush.
  Whole-frame, not subject-specific, and three of the four captures here have a correctly exposed
  backdrop.
- `B-thumbnail-cold-first-frame-no-stats` (DONE, High) — the nearest *mechanism* (missing compile
  wait, wrong frame, healthy payload) and the source of the fix direction above; argued against as
  the cause in this case for the reasons given.
- `B-capture-preview-decoration-not-suppressible` (OPEN, Medium) — same preview rig, different
  axis (decoration suppression). Partly stale: `showFloor` / `showEnvironment` now exist at
  `PreviewSceneRig.cpp:696-711`.

`B-niagara-preview-capture-crashes-editor` does not exist on this board under that id; the nearest
are `B-niagara-edit-with-open-asset-editor-slate-crash` (DONE, Critical) and
`B-niagara-preview-age-zero-after-ticks` (IN-REVIEW, High), neither of which is this.
`B-screenshot-designer-double-srgb` does exist (DONE, High) and is the UMG designer path with a
double sRGB encode — a different verb and a whole-image transform, not a black subject.

## What was NOT done

*(This section records the REPORTER's pass, History `#1`. The detection half has since landed —
see History `#2` and the engine-source section above. The cause is still not established.)*

- No source was modified and no fix was attempted.
- **The cause was not established.** See the candidate list above; the environment/skylight
  candidate is untested and cheap to test.
- The preview-scene readback fields the response already carries (`showEnvironment`,
  `skyIntensity`, `keyIntensity`, `skyVisible`) were **not recorded** from the four captures. That
  is the single most useful missing measurement in this ticket.
- Only foliage subjects were captured. Whether other two-sided or masked materials fail the same
  way is unknown; the class boundary is asserted as "foliage" only because that is what was tested.
- The four in-level renders that establish the assets are fine were made with a different verb;
  no attempt was made to reproduce the black subject in-level under matched lighting, which would
  separate "the preview scene's lighting" from "the preview scene" more sharply than anything above.

severity rationale: impact=High. Two of the rubric's bands are in play and both land here. Primary is the "High or Medium" band, hard blocker with no workaround so a reasonable task is impossible: the verb cannot perform its stated function for an entire asset category, and the task it blocks — look at a plant before planting it — has no substitute *within this verb*. Secondary is the High band's silent wrong output for any caller that gates on the response rather than the pixels: `blank: false` is a true statement about the frame and a false impression of the capture, and nothing in the payload describes the subject separately from the frame, so an automated gate passes a useless image through. NOT Critical: no crash and no write — this is a read-only capture and nothing is corrupted or lost. NOT Medium, and this is the specific reading being rejected: a reviewer can call the in-level route (spawn the asset, frame a camera, `render.capture_open_level`, clean up) a workaround and land on Medium, since it is exactly what proved the assets are fine. It is not a workaround for this verb — it is a different verb producing a different picture, an asset in a level rather than an asset preview, requiring a level, a spawn, a camera solve and a cleanup, and it is documented nowhere as the foliage route. Medium's wording is "doable, but only via a documented workaround, a source dive, or many extra calls"; this is several extra calls AND a different output, and the caller has no signal telling them to reach for it. x reach: BOTH modifiers declined. Not a bump up — `render.capture_asset_preview` is one verb in the render namespace, reached when reviewing an asset, not in almost every session. Not a bump down — vegetation is not a rare edge path: it is one of the largest asset categories in any outdoor scene, it is the category where pre-placement review has the most leverage, and four of four subjects failed on the first attempt. High stands unmodified.

## History
- `#1-foliage-subject-black-in-lit-preview` `OPEN` reporter — Measured live against a running editor. `render.capture_asset_preview` on four foliage assets: `HillTree_P2` returned `blank:false`, mean luminance **0.033**, whole scene black; the other three returned a **correctly lit environment with a pure black subject**. All four render correctly IN-LEVEL, so the assets and materials are fine and the defect is in the preview path. All line numbers re-derived this session at HEAD. **Alpha-zero explicitly ruled out** (the standing false alarm on this project): `blank:false` plus non-zero mean luminance come from the classifier at `PreviewViewportCaptureUtils.cpp:520` computed off the real `ReadPixels` buffer (`:2099`), and an alpha problem cannot selectively black one object while leaving backdrop and floor intact — three of four frames show exactly that. **Verified in source**: the verb builds NO preview scene, it opens the real asset editor and reads its live viewport (`CaptureSubject.cpp:1548-1549`, `:1570-1579`; `PreviewSceneRig.cpp:115`, cast `:124`; renderer tag `PreviewViewportCaptureUtils.cpp:1614`, pixels `:2099`) — so the lights are the engine's `FAdvancedPreviewScene` key + skylight + sky sphere, which PinWright only READS (`PreviewSceneRig.cpp:438-453`) unless `previewScene` is passed (write path `:696-734`, every pin defaults not-provided `PreviewSceneRig.h:76-102`; the one forced default is Niagara `bShowFloor=false`, `RenderHandler.cpp:918-923`). Ruled out in source: `ShowFlags.Lighting` never written (read-only at `PreviewViewportCaptureUtils.cpp:349`; no `SetLightingOnlyOverride` in `Handlers/Render/`), no default view-mode override (`:1411-1416`), no skylight removal, no Lumen/GI manipulation. **Verified GAP, argued against as the cause**: this path waits for NO compilation — `grep -rn` for `FAssetCompilingManager`/`GShaderCompilingManager`/`FinishAllCompilation`/`FinishCompilationForObjects` over all of `Handlers/Render/` returns nothing, while the sibling `asset.generate_thumbnail` blocks at `Handlers/Asset/ThumbnailFrameEvidence.cpp:61`; the only sync is `FlushRenderingCommands()` in `PumpViewport` (`:123`) plus a whole-frame mean-luminance settle loop (`:2145-2172`, `:2168`) that a small subject cannot move. Real defect, worth fixing, but the same assets rendered correctly in-level in the same session (so their shaders were compiled) and four of four failed consistently rather than flakily — so do not expect it to close this ticket. **Cause NOT established.** Strongest untested candidate is the advanced preview scene's `bShowEnvironment` being off, which hides the sky sphere and starves the indirect/transmitted term a two-sided-foliage shading model needs while a directional key still lights opaque surfaces — that matches the signature exactly and the response ALREADY publishes the answer (`PreviewSceneRig.cpp:456-471` measures, `PreviewViewportCaptureUtils.cpp:2799-2801` emits `showFloor`/`showEnvironment`/`rotateLightingRig`, key/sky at `:438-453`); those fields were present in the four captures and were NOT recorded, and re-reading them costs one call. Other candidates, listed not verified: the fixed key-light direction versus a two-sided card, and whether the preview world has Lumen/DFAO/sky occlusion at all (an engine + `AssetViewerSettings` property this plugin never touches). Useful negative: `grep` over `Handlers/Render/` for `two-sided`/`twosided`/`subsurface`/`transmission`/`shading model` returns ZERO hits — nothing in the plugin is foliage-aware, so if the cause is preview lighting state, the fix is to pin it rather than inherit the editor's. NOT DONE: no source modified; cause unestablished; the preview-rig response fields not recorded; only foliage subjects captured, so the class boundary is asserted from what was tested; no attempt to reproduce the black subject in-level under matched lighting. Dedup: `grep -ril capture_asset_preview` across the board returns 22 tickets and none is a subject-black-in-lit-environment claim. `B-capture-preview-default-camera-vs-light` (OPEN, Medium) is closest and explicitly DISCLAIMS this case — its body says its luminance gap is a backdrop-dominated frame mean, "not evidence the subject is better shaded", and names the discriminator it lacked, which this finding supplies; cross-linked, not folded. `B-capture-asset-preview-renders-empty` (IN-REVIEW, High) is the opposite signature (whole frame solid white, non-realtime viewport, already fixed by `AddRealtimeOverride` `:1673`). `B-exposure-pin-black-frame` (IN-REVIEW, High) is a whole-frame exposure crush. `B-thumbnail-cold-first-frame-no-stats` (DONE, High) is the nearest mechanism and the source of the compile-wait fix direction. `B-niagara-preview-capture-crashes-editor` does NOT exist under that id (nearest: `B-niagara-edit-with-open-asset-editor-slate-crash` DONE/Critical, `B-niagara-preview-age-zero-after-ticks` IN-REVIEW/High); `B-screenshot-designer-double-srgb` DOES exist (DONE, High) and is the UMG path's whole-image sRGB double-encode, not this. No umbrella filed; the class statement stays in `B-foliage-paint-does-no-ground-projection` and is referenced.
- `#2-subject-region-measurement-landed-cause-still-open` `OPEN` developer — "Landed the DETECTION half in full and left the ticket OPEN because the CAUSE is still not established; shipping an unverified cause fix is exactly what this ticket warns against. **New: `subjectRegion`**, the only block in a capture response that describes the SUBJECT rather than the frame. New files `Handlers/Render/SubjectRegionStats.h/.cpp` (namespace `PinWrightSubjectRegion`) project the subject's own bounding sphere into the frame the pixels were rendered with and compare the UNLIT SHARE inside that ellipse against the unlit share outside it; the verdict `silhouette` fires only when the subject's disc is >= 5 percentage points more unlit than its backdrop AND the backdrop is >= 50% lit. It is a COMPARISON, never an absolute darkness threshold, so a profile with its environment off (black backdrop) and a genuinely dark preview both refuse the verdict instead of flagging every asset in them; the lit threshold is `PinWrightRenderCapture::BlankLitLuminanceThreshold` passed in rather than redeclared, so `subjectRegion.subject.unlitFraction` and `imageStats.litPixelFraction` can never drift apart. Published fields: `measured` + `notMeasured` reason, `ellipse` {centerX,centerY,radiusX,radiusY}, `subject` {pixelCount,meanLuminance,maxLuminance,unlitFraction}, `backdrop` {the same three}, `litLuminanceThreshold`, `silhouette`, and a sibling `subjectRegionWarning`. `maxLuminance` is the field that answers the ticket's actual question — a darkly SHADED subject still carries a highlight, one that never shaded carries none. Honesty rules per `rpc-design.md` §4: omitted-not-zeroed (no bounds / camera inside bounds / short readback all report `measured:false` WITH a reason rather than a zeroed region that would read as 'the subject is entirely black'), and the warning names what it does NOT establish — a genuinely black asset and one that failed to shade are the same pixels — and points at `viewport.previewScene.showEnvironment` / `sky.visible` / `sky.intensity` / `key.intensity` in the same response. Wired in 5 places: `FViewportCaptureRequest` gains `BoundsOrigin`/`BoundsRadius` and `FViewportCaptureOutput` gains `SubjectRegion` (`PreviewViewportCaptureUtils.h`); measured ONCE on the settled buffer after the warm-up settle and the blank-redraw retry and before the PNG encode, inside `CaptureEditorViewportToPng`, where ReadPixels' buffer is still alive so no frame has to be retained (`PreviewViewportCaptureUtils.cpp`); `PoseListCapture.cpp`'s `MakeFrameRequest` forwards the same bounds `EvaluateBoundsFraming` already gets; emitted by `AddCaptureFields` (`RenderHandler.cpp`) AND by `AddShotFields` (`CameraShotPlanUtils.h`) — both, deliberately, because that file's own comment records the last time a measurement landed on one and not the other. **Deliberately NOT used: the `subjectCoverage` mask**, which would segment the subject exactly; hiding the preview component also removes its SHADOW, so the changed-pixel set is subject + shadow and a shadow is genuinely unlit — segmenting that way reports a correctly shaded asset on a lit floor as substantially unlit, i.e. FALSE POSITIVES, where the bounding-sphere ellipse's dilution gives false negatives. On a verdict whose job is to make a caller distrust an image the error has to point that way; the rationale is in the header so nobody 'sharpens' it later. Regression test `Tests/Render/TestCaptureSubjectRegion.cpp`, 3 ids under `PinWright.render.subject_region.*` (`check_test_ids.py` CLEAN, 4748 ids, no dot-prefix collision). The first test IS the counterfactual and is written the way round that matters: on the SAME synthetic buffer it asserts `blank:false`, mean 0.47, litPixelFraction 0.94 and no tone collapse — every pre-existing criterion reporting a healthy frame — and then that `silhouette` fires anyway. The fixtures are a black band filling only ~a third of its own projected disc (the diluted reading a real bounding sphere gives, not a filled disc), a dark-everywhere frame, and a merely dark-but-shaded subject; the last two are built to trip an absolute threshold and asserted NOT to trip this one. Docs: new `## subjectRegion` section in `Docs/wiki-src/render.md`, placed ABOVE the first `### ` (line 177 vs 243) so it actually renders, plus `subjectRegion` added to the `shots[]` field row. **CAUSE: still unestablished, and three of the ticket's candidates are now REFUTED in engine source** — see the new body section for the citations: `bShowEnvironment` cannot starve the skylight (`PreviewScene.cpp:92` `SLS_SpecifiedCubemap`; `AdvancedPreviewScene.cpp:225` moves only the sky SPHERE, while `:229` shows the skylight's switch is the separate `bUseSkyLighting`, which PinWright ALREADY reports as `sky.visible` from `PreviewSceneRig.cpp:449`); the preview key light already carries the transmission term two-sided foliage needs (`PreviewScene.cpp:87` `bTransmission = true`); and the compile-wait candidate predicts a GREY subject rather than a black one, because an incomplete shader map falls back to the default material (`MaterialRenderProxy.cpp:865-888`). One new fact for whoever adds the compile wait anyway: `PumpViewport` (`PreviewViewportCaptureUtils.cpp:106-125`) drives Slate + Draw + FlushRenderingCommands and NEVER `FAssetCompilingManager::ProcessAsyncTasks`, so the settle loop cannot advance an in-flight compile however long it spins — the wait must call `PinWright::AssetCompile::AdvanceOnGameThread()` or `FinishCompilationForObjects`, not pump harder. The compile wait was NOT added: the evidence now argues against it as this cause, it is a behaviour change on the hot path of every capture verb, and bundling it would be the speculative fix this ticket exists to refuse — file it separately. NOT DONE: no live capture was taken (the gateway refused with `EDITOR_NOT_RUNNING` and the checkout was mid-edit by 27 other agents), so `showEnvironment` / `skyIntensity` / `keyIntensity` / `skyVisible` are STILL unrecorded for the four subjects and the new `subjectRegion` block has never been exercised against real pixels — only against synthetic buffers. Nothing was compiled or run; the orchestrator builds. **Next step is one call, and it is now cheaper than before:** re-capture one foliage asset and read `subjectRegion.subject.maxLuminance` beside the four preview-scene fields — max luminance at the floor says 'never shaded', a real highlight says 'shaded dark', and that single number splits the remaining candidate space."
