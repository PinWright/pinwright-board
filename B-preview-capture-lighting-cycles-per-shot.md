---
id: B-preview-capture-lighting-cycles-per-shot
title: "Consecutive asset-preview captures of ONE stationary pose return a different shading solution — a deterministic period-4 cycle of +-14/255 on the subject, indexed by capture number and invisible in every published signal, while the previewScene parameter's own wire documentation promises the rig is applied once for the whole set"
status: OPEN
severity: High
category: bug
tags: [render, camera, orbit_shots, capture_asset_preview, previewScene, preview-scene, rig, warmup, settle, nondeterministic, lighting, convergence, silent-wrong-data, reproduced, comparability, doc-contradicts-code]
encounters: 1
lastSeen: 2026-09-02T20:45:00+03:00
---
# One pose, three captures, three different shading solutions — and every published signal reads healthy

**Reproduced, not relayed.** The observation came from the rendering agent; I re-derived every
number below by decoding the 18 delivered PNGs off disk with a stdlib decoder, and confirmed the
whole-frame means against the response's own `imageStats.meanLuminance` to four decimal places.
The mechanism section is read from source at plugin HEAD `47307435`.

## What was measured

`camera.orbit_shots` — the verb is established from the filenames, which only this verb writes
(`FilenamePrefix`/`Subdirectory` = `"CameraOrbit"`, `CameraFrameHandler.cpp:1130-1131`) —
`subject {kind:"staticMesh", path:"/Game/Atlantis/Meshes/SM_Column_Doric"}`,
`previewScene {showFloor:true, showEnvironment:true}`, `exposure {mode:"fixed", ev100:-2.2869}`,
1120x1400, **18 shots in ONE call: 6 azimuths (144, 145.5, 147, 148.5, 150, 151.5) at elevation 8,
each captured three times in a row with the camera stationary.**

Response:
`Saved/PinWright/HttpResponses/20260902T092641Z/20260902T171435Z_388389c9-4dd5-8056-c671-18ab7770a275.json`.
Stills: `Saved/Screenshots/CameraOrbit/CameraOrbit_20260902_201429_shot00..17_*.png`
(1120x1400, 8-bit RGBA, alpha uniformly 255 — no alpha confound).

**Every published health signal is green on all 18 shots.** `warmup {settled:true, settleRounds:1,
meanLuminanceDelta:2.0808e-5, settleMs:33.37}`; `exposure {pinned:true, pinnedFrameUsable:true,
fixed:true, adapted:4.880064, adaptedSource:"fixedPin", restored:true}`;
`previewScene {requested:true, applied:true, restored:true, restore.profilesRestored:true,
restore.configFileUnchanged:true}`; `blank:false`, `litPixelFraction:1.0` on every shot;
`poseSet {posesRequested:18, posesCaptured:18, posesTruncated:0, posesOutOfFrame:0}`; no `warnings`.

**The pixels disagree.** Rec.709 luma, 0-255:

| comparison | mean abs diff | max abs | px differing | px >10/255 |
|---|---|---|---|---|
| same pose, adjacent repeats (18 pairs) | 0.52 - 0.75 | 45 - 69 | 65.5% - 69.3% | 0.07% - 0.76% |
| **1.5 deg pose change (4 pairs)** | **13.08 - 13.29** | 147 - 148 | 96.5% - 96.9% | **32.9% - 33.7%** |

No same-pose pair is identical; the smallest differs in 1,036,331 of 1,568,000 pixels.

## The signal is a period-4 cycle indexed by capture number, on the subject

A 28x28 patch at pixel `(504,1064)` — the column base plinth's front face, confirmed by cropping and
looking at it — over the 18 shots **in capture order**:

    177.05 162.04 149.07 162.14 | 176.65 162.18 149.24 162.13 | 176.83 161.69 149.27 162.48
    176.92 161.89 149.25 162.50 | 176.57 162.45

| phase (index mod 4) | n | mean | sd |
|---|---|---|---|
| 0 | 5 | 176.805 | 0.176 |
| 1 | 5 | 162.050 | 0.259 |
| 2 | 4 | 149.210 | 0.080 |
| 3 | 4 | 162.310 | 0.178 |

Variance explained by period 4 = **1.000**; by periods 2/3/5/6/7/9 = 0.015/0.046/0.027/0.114/0.238/0.484.
Linear trend against capture index R^2 = **0.000**. Phase scatter (sd 0.08-0.26) is ~100x below the
27.98 amplitude, so the cycle is **deterministic, not noise**. On this ROI the capture-index swing
(27.98) is **3x the azimuth effect** (9.36 across the six triplet means) — the frame changes more
from being taken again than from being taken from somewhere else.

**It is a shading change, not anti-aliasing jitter.** Three independent tests, all in the same
direction:

1. **84%-93% of significant differences fall OFF high-gradient edges.** Edges (local gradient >20/255)
   are 0.53% of the frame; they are enriched 14-30x but still carry a small minority of the change.
2. **Flat-interior signed means are non-zero and phase-coherent** — +0.25 / -0.19 (0-255), flipping
   sign with the period-4 phase. Jitter averages to zero over flat regions.
3. **The difference image is filled faces, not rims.** An 8x-amplified signed diff shows solid
   rectangles over the plinth's entire front face and the whole lower column shaft plus a broad
   diffuse darkening of the ground around the base. A 3x-zoom strip of shots 00/01/02/03 steps
   bright -> mid -> clearly darkest -> mid **uniformly across the face** while the camera has not
   moved. The change is visible to the eye.

Separately and harmlessly, a **1-LSB dither floor** covers the whole frame: the sky region's max
per-pixel difference is exactly 1.0 with a signed mean of 0.0006. That floor is consistent with the
anti-aliasing story in `B-capture-render-resolution-unreported`; the +-14/255 localised swing is not,
and the two must not be collapsed (see *Related*).

**Nothing published can see it.** The whole-frame mean moves ~0.001 (normalised) inside a triplet —
the plinth is a small fraction of 1,568,000 pixels, so a 28/255 swing on it disappears into the
average. `imageStats` publishes exactly that mean plus variance, min, max and `litPixelFraction`;
`subjectRegion.subject.meanLuminance` is an average over the whole silhouette and moves ~0.003.
There is no field on any capture verb that would come back different.

## Mechanism, read from source

`camera.orbit_shots` (`CameraFrameHandler.cpp:678`) builds poses and hands them to the shared
primitive: `PinWrightPoseCapture::CaptureCameraPoses` -> `RunPoseListCapture`, which calls
`CaptureFrame` **once per pose** (`PoseListCapture.cpp:300`) plus one discarded warm-up
(`:181`). `CaptureFrame` calls `CaptureEditorViewportToPng`
(`PreviewViewportCaptureUtils.cpp:1643`). `camera.orbit_shots` binds no
`SubjectVisibilitySetter` and no coverage flag (grep: zero hits in `CameraFrameHandler.cpp`), and
the response carries no `subjectCoverage`, so there is exactly **one** capture per shot — 19
`CaptureEditorViewportToPng` calls for this 18-shot set.

**Nothing on that path waits for, drives, or suppresses any temporal or global-illumination
accumulation.** A grep of `PreviewViewportCaptureUtils.cpp/.h` for
`Lumen|DistanceField|DFAO|SkyLight|RealTimeReflection|Recapture` returns **two comment hits and no
code** (`PreviewViewportCaptureUtils.h:56`, `:1471`); `RecaptureSky` exists only in
`Handlers/Environment/LightingHandler.cpp:526,734`, off this path. Temporal AA is never suppressed.
The only show flags this path writes are `EyeAdaptation` off while an exposure pin is in force
(`:240`, restored `:252`), billboard sprites (`:1349`), and a view-mode override (`:1835`).

Per shot the frame is pumped `RequestRealTimeFrames(2)` + **3 unconditional pumps**
(`:2192-2196`), read (`:2218`), then the settle loop adds **at least one and at most eight** more
(`:2246-2280`). `PumpViewport` (`:107-125`) is Slate `PumpMessages` + `Tick` + two `Invalidate`s +
`SceneViewport->Draw()` + `FlushCommands()` + `FlushRenderingCommands()`. So the floor is four
draws per shot, and with `settleRounds:1` reported that is what this set paid.

**Two per-shot state churns are established, and both are candidate carriers. Neither is proven to
be the cause — the period-4 structure is not derived below, it is measured.**

### 1. The preview-scene rig is applied and torn down around EVERY shot, and the parameter's own wire documentation says otherwise

`FScopedPreviewSceneRig` is constructed at **exactly one site in the whole tree** —
`PreviewViewportCaptureUtils.cpp:1906` — which is inside `CaptureEditorViewportToPng`, i.e. **per
shot**. Its constructor writes `Profile->bShowFloor` + `SetFloorVisibility(..., bDirect=true)` and
`Profile->bShowEnvironment` + `SetEnvironmentVisibility(..., bDirect=true)`
(`PreviewSceneRig.cpp:700-709`). Its destructor restores the key light's direction, brightness and
colour and the sky light's brightness (`:751-766`), restores the shared profile array, and
re-asserts floor and environment visibility from the restored profile (`:787-788`). So on an 18-shot
set the preview scene's lights and visibility flags are written **38 times**, half of them putting
the scene back to a state the caller did not ask for, between every pair of shots.

Three places in this tree state the opposite, and one of them is **on the wire**:

- `CameraFrameHandler.cpp:706`, the `previewScene` parameter description an agent reads before
  calling: *"Read once for the whole set, so every shot in one call is lit identically, and
  restored once after the last one."*
- `CameraFrameHandler.cpp:774-777`: *"Applied and restored ONCE around the whole set by the capture
  primitive's scoped guard."*
- `CameraFrameHandler.cpp:1135-1136`: *"Set-level like the exposure pin: one rig applied and
  restored once around the whole orbit, **so no two shots in one set can be lit differently from
  each other or from their own report**."*

That last sentence is the exact invariant this ticket reports broken, asserted as a guarantee.

**The response proves the teardown at the wire level, without any source reading.**
`viewport.previewScene` is reported once from the last capture of the set. On this call it reads
`showFloor: true` (applied) with **`previous.showFloor: false` and `afterRestore.showFloor: false`**.
The caller asked for `showFloor: true` for the whole set; if the rig were set-level the floor would
already have been on when the last shot began. It was off, because the previous shot's destructor
had turned it back off.

Note the asymmetry with the neighbouring pins. `exposure`, `hideEditorSprites` and `viewMode` are
also applied per shot (`:240`, `:1349`, `:1835`) and their "read once for the whole set" wording is
equally inaccurate — but they write **view state** with the same value every time, so repetition is
inert. The rig writes **preview-scene component state** and actively reverts it, so repeating it is
not a no-op even when the value never changes.

### 2. The render target is unfixed and re-fixed around every shot

`SceneViewport->SetFixedViewportSize(Request.Width, Request.Height)` at `:1768`, and the
`ON_SCOPE_EXIT` calls `SetFixedViewportSize(0, 0)` at `:1731`. Between shot N and shot N+1 the
viewport therefore returns to its natural size and is resized back, so any history buffer keyed to
the render-target size (temporal AA history, Lumen/DFAO temporal filters, eye-adaptation state) is
torn down and rebuilt once per shot rather than persisting across the set.

**A constraint on any diagnosis, from the data:** a period of exactly 4 in the capture index means
the carrying state **survives across captures** and advances a fixed amount per capture. A term that
were fully reset per shot would make every shot identical; one that were converging would show a
monotone trend. Neither is what the numbers do.

## Severity

**High.** Impact class is the rubric's *"silent wrong data on a normal path — the caller trusts a
result that is a lie and builds on it"*. Every field the response publishes to certify a capture
reads healthy — `settled`, `pinned`, `applied`, `restored`, `blank`, `litPixelFraction`,
`boundsInFrame`, no warnings — over frames whose subject differs by up to 28/255 for no reason the
caller can name or control. The lie is not in any single field; it is in the set of them being
exhaustive, which is what an agent has to assume when it decides whether two frames are comparable.

Reach argues up rather than down: the affected primitive `CaptureEditorViewportToPng` sits under
every capture verb in the plugin, and the failing population is **every asset-subject capture set** —
the pose-set verbs exist precisely to produce comparable frames, so this defeats their purpose
rather than degrading it. Field cost, same session: a 240-still turntable was unusable, with 34
frame-to-frame luminance steps above 3/255 and a largest of 56/255 (**relayed**, not re-measured
here), while every published number said the set was healthy.

Not Critical: no crash and no asset write, so it stays under the Critical band's two triggers.

**Workaround:** none that removes the effect. Partial mitigations, all costly: take one shot per
call and discard the first; or capture 4k shots per pose and average phases; or accept only
comparisons between captures whose index is congruent mod 4, which is not knowable from the response
because no phase is published.

## Ask

1. **Publish a convergence measurement, not just a frame-mean settle.** The pieces already exist.
   `PinWrightPoseCapture::MeasureChangedPixelFraction` (`PoseListCapture.cpp:502-541`, declared
   `PoseListCapture.h:368`) is a per-pixel channel-threshold diff already used per shot for the
   subject-coverage differential (`:316`), and the request can already retain pixels
   (`Frame.bRetainPixels`, `:297`). Compare the settled frame against the previous settle round's
   buffer and publish the changed-pixel fraction and max per-pixel delta beside
   `meanLuminanceDelta`. A whole-frame scalar mean cannot see a localised swing (see
   `B-preview-rig-first-capture-stale-sky` `#2`/`#3`); a per-pixel measure can.
2. **Hoist the preview-scene rig to the set, and make the docs match wherever it lands.** Construct
   `FScopedPreviewSceneRig` once around `RunPoseListCapture` instead of inside
   `CaptureEditorViewportToPng`, which is what `CameraFrameHandler.cpp:706`, `:774-777` and
   `:1135-1136` already claim. If per-shot scoping must stay for a reason not visible here, then all
   three of those texts are wrong and the wire one must be corrected first — it is the one a caller
   plans against.
3. **Give the set a same-pose control shot.** The primitive already takes one throwaway warm-up
   frame per call (`PoseListCapture.cpp:150-197`). Re-shoot pose 0 at the end of the set and publish
   the diff against the original as `poseSet.poseRepeatability {meanAbsDelta, maxDelta,
   changedPixelFraction}`. That is one extra capture per set and it converts "are these frames
   comparable?" from an assumption into a measurement — the same argument
   `ThumbnailFrameEvidence.h:43-46` already makes for the thumbnail path.
4. **Model to copy for a gate, if one is added:** the thumbnail path's render-twice-and-compare,
   gated on a measurement and bounded at one extra pass (`AssetWorkflowHandler.cpp:978-990`,
   `ThumbnailFrameEvidence.cpp:131` `FrameIsDegenerate`, `:145` `CountDifferingPixels`); and the
   consecutive-stable-ticks pattern (`DriveSettleDecision.cpp:60-67`, default 2 in
   `Handlers/Drive/DriveTypes.h:209`) — the viewport settle loop requires only **one** stable pair,
   so a frame that momentarily plateaus exits immediately.

## Related

- `B-capture-render-resolution-unreported` (OPEN/High) — **the ticket this is most likely to be
  merged into, and it should not be.** That one reports shot-to-shot differences at one pose across
  240 stills and attributes them to unsuppressed TSR jitter, with `settleRounds:8,
  meanLuminanceDelta 2.7e-3`. This set is the opposite signature — `settleRounds:1`,
  `meanLuminanceDelta 2.08e-5` — and the three tests above **rule out** jitter as the carrier of the
  +-14/255 swing: off-edge concentration, non-zero phase-coherent flat-region signed means, and
  filled-face rather than rim-shaped difference images. The 1-LSB whole-frame dither floor measured
  here (sky max diff exactly 1.0) is plausibly that ticket's effect; the localised swing is a second,
  larger signal on the same verb. Its Ask 3 ("suppress temporal AA for a still, or report that it
  was not") would not remove this one.
- `B-preview-rig-first-capture-stale-sky` (OPEN/Medium) — same verb, same subject, same day, and it
  owns the *first-vs-second capture* transient plus the invariant that no capture call runs the
  editor tick which completes preview-scene work. This is the steady-state complement: the cycle
  persists indefinitely rather than clearing on the next call. Its Ask 1 cites
  `CameraFrameHandler.cpp:706` as an existing "apply once for the whole set" seam — that seam does
  not exist, and the correction is recorded in that ticket's history.
- `B-capture-asset-preview-renders-foliage-black` (IN-REVIEW/High) — its `#1` names "whether the
  preview world has Lumen / DFAO / sky occlusion at all" as an untested candidate, then closes the
  candidate space on exposure. That question is still unanswered and is the first thing to check
  here.
- `B-mrq-artifact-report-omits-stream-count` `#2`, `B-ground-probe-hits-hull-not-render` (DONE/High),
  `B-thumbnail-cold-first-frame-no-stats` (DONE/High) — the recurring shape: the call succeeds,
  every reported number is correct, and the output is wrong because the number that mattered was
  never reported.

## History
- `#1-period-4-shading-cycle-reproduced` `OPEN` reporter — **Reproduced from the delivered pixels, not relayed.** `camera.orbit_shots` (verb established from the `CameraOrbit` filename prefix, written only at `CameraFrameHandler.cpp:1130-1131`), staticMesh `/Game/Atlantis/Meshes/SM_Column_Doric`, `previewScene {showFloor:true, showEnvironment:true}`, `exposure {mode:"fixed", ev100:-2.2869}`, 1120x1400, 18 shots in one call = 6 azimuths (144/145.5/147/148.5/150/151.5, el 8) x 3 repeats of the identical pose. Response `Saved/PinWright/HttpResponses/20260902T092641Z/20260902T171435Z_388389c9-…json`; stills `Saved/Screenshots/CameraOrbit/CameraOrbit_20260902_201429_shot00..17_*.png`, decoded with a stdlib PNG decoder whose whole-frame means reproduce the response's own `imageStats.meanLuminance` to 4 dp. FINDINGS: no same-pose repeat is identical — 65.5%-69.3% of the 1,568,000 pixels differ, mean abs 0.52-0.75, max 45-69/255 — against 13.08-13.29 mean abs for a real 1.5-degree move, so a repeat scores ~1/18 of a pose change but is not zero. The difference is sharply localised: a 28x28 patch at (504,1064), confirmed by cropping to be the base plinth's front face, reads 177.05/162.04/149.07/162.14/176.65/162.18/149.24/162.13/176.83/161.69/149.27/162.48/176.92/161.89/149.25/162.50/176.57/162.45 in capture order — **period exactly 4 by capture index, variance explained 1.000 (p2 0.015, p3 0.046, p5 0.027, p6 0.114, p7 0.238, p9 0.484), linear-trend R^2 vs index 0.000, phase sd 0.08-0.26 against a 27.98 amplitude**, i.e. deterministic; on that ROI the capture-index swing is 3x the azimuth effect (27.98 vs 9.36). NOT anti-aliasing jitter, on three tests: 84%-93% of >3/255 differences fall off high-gradient edges (edges are 0.53% of the frame), flat-interior signed means reach +0.25/-0.19 and flip sign with the phase, and the 8x-amplified signed diff shows filled faces and a diffuse ground darkening rather than silhouette rims — visually confirmed on a 3x zoom strip where the plinth face steps bright/mid/darkest/mid across three captures of one camera pose. A separate 1-LSB dither floor covers the whole frame (sky max diff exactly 1.0, signed mean 0.0006) and is plausibly `B-capture-render-resolution-unreported`'s effect; the two signals must not be collapsed. INVISIBLE TO EVERY PUBLISHED FIELD: `warmup {settled:true, settleRounds:1, meanLuminanceDelta:2.0808e-5}`, `exposure {pinned:true, adapted:4.880064, adaptedSource:"fixedPin"}`, `previewScene {applied:true, restored:true}`, `blank:false`, `litPixelFraction:1.0`, no warnings; the whole-frame mean moves ~0.001 normalised inside a triplet. MECHANISM (source, plugin HEAD `47307435`): one `CaptureEditorViewportToPng` (`PreviewViewportCaptureUtils.cpp:1643`) per pose (`PoseListCapture.cpp:300`) plus one warm-up (`:181`); `camera.orbit_shots` binds no coverage/visibility setter so there is no second draw per shot; nothing on the path waits for or suppresses Lumen/DFAO/skylight/temporal AA (grep of the capture utils for `Lumen|DistanceField|DFAO|SkyLight|RealTimeReflection|Recapture` yields two comments and no code); 3 unconditional pumps (`:2192-2196`) then a settle loop of 1-8 (`:2246-2280`). **Two established per-shot churns, neither proven to be the carrier:** (1) `FScopedPreviewSceneRig` is constructed at exactly one site tree-wide, `PreviewViewportCaptureUtils.cpp:1906`, i.e. per shot — ctor writes floor/environment visibility (`PreviewSceneRig.cpp:700-709`), dtor restores key-light direction/brightness/colour, sky brightness and floor/environment (`:751-766`, `:787-788`), so an 18-shot set writes preview-scene state 38 times; (2) `SetFixedViewportSize(W,H)` at `:1768` with `SetFixedViewportSize(0,0)` on scope exit at `:1731`, so the render target is unfixed and re-fixed between every pair of shots. **The set-level claim is false and it is on the wire:** `CameraFrameHandler.cpp:706` (the `previewScene` parameter description) says "Read once for the whole set, so every shot in one call is lit identically, and restored once after the last one", `:774-777` says "Applied and restored ONCE around the whole set", and `:1135-1136` says "one rig applied and restored once around the whole orbit, so no two shots in one set can be lit differently from each other or from their own report" — the exact invariant this ticket reports broken. Proven from the response alone: `viewport.previewScene` (reported from the last capture) has `showFloor: true` applied with `previous.showFloor: false` and `afterRestore.showFloor: false`, so the floor was off entering the last shot of a set that requested it throughout. A period of exactly 4 constrains any diagnosis: the carrying state survives across captures and advances per capture, so it is neither fully reset per shot nor converging. Severity High: silent wrong data on a normal path with every certifying field green, reach argued up because `CaptureEditorViewportToPng` is under every capture verb and the failing population is every asset-subject comparison set; not Critical (no crash, no asset write). Field cost RELAYED and not re-measured here: a 240-still turntable unusable, 34 steps above 3/255, largest 56/255.
