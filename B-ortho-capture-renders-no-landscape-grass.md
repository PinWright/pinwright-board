---
id: B-ortho-capture-renders-no-landscape-grass
title: "An orthographic capture renders zero landscape grass at a frame the perspective capture fills with it, and distance culling is measurably not the cause — cullingOriginPushback 0, derived r.ViewDistanceScale 2.29, unlit, still no grass"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture_open_level, capture_ortho_tiles, orthographic, landscape, grass, vegetation, unattributed-mechanism, silent-wrong-data, verification-evidence]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Ortho renders the terrain and none of the grass on it

At matched world coverage, an orthographic capture of a grassed landscape contains **no grass at
all**, while the perspective capture of the same ground contains a full carpet. The terrain,
lighting and everything else render; only the landscape grass is missing, and it is missing
entirely rather than thinning toward the frame edges.

## Measured, matched coverage

`orthoWidth 6928` was chosen to match the world span of `fov 60` at Z 6000, so the two frames cover
the same ground:

| projection | `meanLuminance` | grass in frame |
|---|---|---|
| perspective, fov 60, Z 6000 | 0.4797 | full carpet |
| orthographic, `orthoWidth 6928`, same ground | 0.5161 | **none** |

Direction, stated because the arithmetic misleads: grass darkens the frame, so the *higher* number
is the one with *less* grass. The ortho frame is brighter because the ground under the missing
grass is brighter.

## Distance culling is ruled out by measurement, not by argument

`B-ortho-capture-culls-distant-foliage` (OPEN, High, `encounters: 4`) is the board's wide-ortho
distance-culling defect, and it is the first thing a reader will reach for. It does not explain
this frame:

- With `viewMode: "unlit"` the capture's own survey reported **`cullingOriginPushback: 0`** — the
  field the culling fix added precisely to expose the editor-ortho view-origin pushback that ticket
  `#4` identified as the residual defect (`Handlers/Render/PreviewViewportCaptureUtils.cpp:1983`
  computes it, `:2341` publishes it as `survey.cullingOriginPushback`; declared at
  `PreviewViewportCaptureUtils.h:589`). Zero pushback means the ~2.1e6 uu offset that ticket blames
  is not present in this frame.
- The auto-derivation ran and applied a **derived `r.ViewDistanceScale` of 2.29**, so the
  view-distance override was live, not skipped.
- The grass was still completely absent.

- The framing is also the wrong shape for that ticket. `B-ortho-capture-culls-distant-foliage` is
  about a *wide* frame (its evidence is `orthoWidth 64000` from Z 60000, corners ~74000 cm out)
  losing distant instances while nearer ones survive — a partial, distance-graded loss. This frame
  is `orthoWidth 6928`; its corners are a few thousand cm from centre, and the loss is total.

## The mechanism was not attributed

**The zone agent could not attribute the mechanism.** No cause is proposed here. What is
established is the measurement above and the ruling-out above; anything past that would be a guess,
and a confident wrong cause in a ticket is worse than an honest gap.

Specifically **not** claimed, and left for whoever picks this up:

- whether the grass instances exist and are not drawn, or were never built for this view;
- whether an orthographic editor view contributes a usable entry to
  `World->ViewLocationsRenderedLastFrame` at all;
- whether any show flag or projection-dependent path in the grass renderer is involved.

A sibling ticket filed the same session, `B-capture-open-level-pose-params-photograph-stale-grass`,
does attribute a camera-keyed staleness mechanism for the *perspective* path. **That attribution is
cross-referenced only and is explicitly not offered as this ticket's cause** — it was not tested
against an orthographic view, and this frame's grass is absent even at the pose the persistent
camera is already sitting at.

## Consequence

`render.capture_ortho_tiles` (`Handlers/Render/OrthoTileCaptureHandler.cpp:162`) cannot verify
grass at all. That removes the only tiling path for large-area vegetation review, and it removes it
*silently*: the tiles come back valid, non-blank and settled, showing ground that in the perspective
capture is covered in grass.

It also collides with the board's own prescription. `B-ortho-capture-culls-distant-foliage` `#5`
records tiled orthographic capture as the cause-level workaround for wide-frame foliage loss —
"capture per tile rather than whole-map". For landscape grass specifically, the prescribed
workaround produces an empty frame, so the two tickets currently point a caller in a circle.

**Workaround:** perspective captures only for any question about landscape grass. There is no
orthographic path.

## Fix

Unattributed, so no implementation is proposed. The first useful step is diagnostic and is worth
recording as the ask: from an orthographic capture, report whether landscape grass components
exist in the scene and how many of their instances are in the frustum, so the next investigator can
tell "not built" from "built and not drawn" without guessing. The capture path already surveys
primitives for the view-distance work (`PreviewViewportCaptureUtils.cpp:1983` and the survey block
published at `:2341`), so the walk is already there to extend.

## Same shape as

- `B-ortho-capture-culls-distant-foliage` (OPEN, High) — **the ticket this one rules out**, not a
  duplicate of it. See the section above for the three measurements that separate them. Filed
  separately rather than as a fifth encounter because appending it there would attach evidence that
  contradicts that ticket's mechanism to that ticket's mechanism, and would let the culling fix
  close a defect it does not touch.
- `B-horizontal-orthographic-views-render-no-geometry` (DONE) — horizontal (`elevation 0`)
  orthographic frames that rendered overlays and no scene at all. Distinct: that was every
  primitive missing on the four side views, fixed; here the top-down view renders the whole scene
  correctly *except* landscape grass.
- `B-capture-open-level-pose-params-photograph-stale-grass` (OPEN) — grass absent from a capture,
  perspective, mechanism attributed. Cross-reference only.

## Severity

**High**, by impact class: *silent wrong data on a normal path*. The ortho frame is not a refusal
and not a blank — it is a plausible, well-formed picture of ground that is actually covered in
grass, and the response has no field that hints otherwise. A reviewer reading it concludes the
vegetation is missing.

**Reach modifier declined.** Orthographic capture is not an every-session path, which by the rubric
would bump this down to Medium. Declined on two grounds: it is the board's own prescribed workaround
for the sibling wide-ortho foliage defect, so it sits on a recommended path rather than an edge
one; and the failure is silent rather than an error, which is the property the High band is keyed
on.

Not rated Critical: nothing is written, nothing is lost, and the editor stays up.

## History
- `#1-ortho-renders-zero-grass-culling-ruled-out` `OPEN` reporter — Measured live against a running
  editor at matched world coverage (`orthoWidth 6928` against `fov 60` at Z 6000): perspective
  `meanLuminance` 0.4797 with a full grass carpet, orthographic 0.5161 with none. Distance culling
  ruled out by measurement rather than argument: with `viewMode: "unlit"` the response reported
  `cullingOriginPushback: 0` and a derived `r.ViewDistanceScale` of 2.29, and the grass was still
  absent; the frame is also two orders of magnitude narrower than the wide-frame case
  `B-ortho-capture-culls-distant-foliage` documents, and the loss is total rather than
  distance-graded. Field citations re-derived: `PreviewViewportCaptureUtils.cpp:1983` computes
  `CullingOriginPushback`, `:2341` publishes it, `PreviewViewportCaptureUtils.h:589` declares it;
  `OrthoTileCaptureHandler.cpp:162` registers `render.capture_ortho_tiles`. **Mechanism not
  attributed** — no cause proposed, three specific unknowns listed in the body.
- `#2-ortho-is-structurally-never-a-grass-camera` `IN-REVIEW` developer — **Mechanism attributed
  from UE 5.8 source, and the ticket's unknown #2 answered: an orthographic editor view contributes
  NOTHING to `World->ViewLocationsRenderedLastFrame`, ever.**
  `UEditorEngine::UpdateSingleViewportClient` gates BOTH camera feeds on
  `InViewportClient->IsPerspective()` — the streaming-manager registration at
  `C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\EditorEngine.cpp:2613-2618` and the
  `ViewLocationsRenderedLastFrame.Add(InViewportClient->GetViewLocation())` at `:2628-2634` — and
  the only other writer of that array, `AddStreamingViewInfo`
  (`Runtime\Engine\Private\UnrealClient.cpp:1771-1787`), is called solely from
  `UGameViewportClient::Draw` (`GameViewportClient.cpp:1913`), which no editor viewport reaches.
  `ULandscapeSubsystem::Tick` feeds `ALandscapeProxy::UpdateGrass` from exactly those two sources
  (`Runtime\Landscape\Private\LandscapeSubsystem.cpp:729-757`, `:891-901`) and skips the call
  entirely when neither yields a camera; the default `grass.UseStreamingManagerForCameras` is **1**
  (`:65-69`), which is the branch with **no** `OldCameras` fallback, so `CurrentViewInfos`
  (rebuilt from scratch each `UpdateResourceStreaming`) simply comes back empty. Grass is a
  BUILD-time decision keyed on those camera points (`LandscapeGrass.cpp:2917-2928`, `:2998-3009`,
  `:3121`), not a render-time cull, and it is scaled by `grass.CullDistanceScale`, **not** by
  `r.ViewDistanceScale` — which is why the derived 2.29 in `#1` could not have helped and why the
  ruling-out in `#1` stands. **So the grass in an orthographic frame is a leftover of the last
  PERSPECTIVE view, and over ground no perspective view has visited the frame is bare.** A
  `USceneCaptureComponent2D` is neither a perspective editor viewport nor a game viewport, so
  `render.capture_ortho_tiles` had the same exposure and no path to grass at all.
  **Same root cause as `B-capture-open-level-pose-params-photograph-stale-grass`**, one step
  further out: that ticket found the POSE never reaches the grass builder; this one finds the
  PROJECTION structurally cannot. Its landed `PinWrightCaptureGrass::SettleGrassForCapturePose`
  (`Handlers/Render/LandscapeGrassSettle.{h,cpp}`, forcing
  `ULandscapeSubsystem::RegenerateGrass(false, /*sync*/true, {effectiveEye})`) sits on the shared
  `render.capture_open_level` path and therefore already covers the orthographic projection —
  **not duplicated here**. Left to this ticket, and done: (a) the same forced build wired into the
  tile path, **per tile**, in `OrthoTileCaptureUtils.cpp` `CaptureTile` before `CaptureScene()`
  (grass is built inside a cull band around one camera point, so a burst settled at tile (0,0)
  still renders bare at (3,3)); `render.capture_ortho_tiles` now publishes a `grass` block on both
  the response and the manifest, with `buildMsTotal` summed over the burst and `allTilesSettled`
  ANDed. (b) The diagnostic this ticket's own Fix section asked for — "report whether landscape
  grass components exist in the scene and how many of their instances are in the frustum, so the
  next investigator can tell 'not built' from 'built and not drawn' without guessing":
  `MeasureGrassFrameReach` walks the grass components from the frame's **measured**
  `FSceneView::CullingOrigin` (reused from the view-distance survey where it ran, else
  `MeasureCaptureCullingOrigin`) and publishes `grass.reach` — `cullingOrigin`,
  `cullingOriginMeasured`, `cullingOriginPushback`, `nearestComponentDistance`,
  `grassCullDistance` (the component's own `InstanceEndCullDistance` × the **effective**
  `r.ViewDistanceScale`), `componentsInReach`, `instancesInReach` — and **warns** when grass is
  built, settled, and none of it can reach the frame. That is the case that shipped silently: a
  plausible, non-blank, settled picture of covered ground with `builtForPose: true` and no field
  that could contradict it. The block is omitted rather than zeroed when unmeasured.
  Regression tests: `Private/Tests/Render/TestCaptureGrassFrameReach.cpp`, new id group
  `PinWright.render.grass_reach.*` (checked against every existing `PinWright.render.*` leaf — no
  shadowing). Five tests: the empty-forest warning fires and names culling rather than absence; a
  frame whose grass is in range warns nothing and does not swallow the unfinished-build warning;
  an unmeasured reach publishes no zeros; an **orthographic** `render.capture_open_level` over an
  in-code `landscape.create` fixture drives its own grass build and reports the same `instances` as
  the perspective capture at the identical pose (the ticket's own A/B, at
  `fov 60` / `orthoWidth 6928` from Z 6000, read off counts instead of luminance); and
  `render.capture_ortho_tiles` publishes the block at all. **A pixel assertion was deliberately not
  written** — see the "what remains" note below.
  **What remains, and it is not this ticket's mechanism.** With the build fixed, a **lit**
  orthographic frame can still show no grass: instances are culled against
  `InstanceEndCullDistance` (default 10000 cm) measured from `FSceneView::CullingOrigin`, which a
  lit editor ortho view pushes ~2.1e6 cm off the camera. That is
  `B-ortho-capture-culls-distant-foliage` `#4`'s pushback, it is that ticket's to fix, and grass is
  additionally immune to its `r.ViewDistanceScale` remedy on the BUILD side because the build scales
  by `grass.CullDistanceScale`. Until that lands, a lit ortho grass frame is disclosed rather than
  recovered — which is why the regression test asserts the warning and the counts rather than
  grass pixels. Not compiled and not run: written by an agent without a build.
