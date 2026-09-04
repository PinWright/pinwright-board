---
id: B-preview-capture-lighting-cycles-per-shot
title: "Consecutive asset-preview captures of ONE stationary pose return a different shading solution — a deterministic period-4 cycle of +-14/255 on the subject, indexed by capture number and invisible in every published signal, while the previewScene parameter's own wire documentation promises the rig is applied once for the whole set"
status: IN-REVIEW
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
The original mechanism evidence below is a pre-#2 record read from plugin commit `47307435`;
its line references, call counts, and behavior are historical, not current-source claims.

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

**Pre-#2 observation (plugin commit `47307435`):** Nothing published could see the cycle. The
whole-frame mean moves ~0.001 (normalised) inside a triplet —
the plinth is a small fraction of 1,568,000 pixels, so a 28/255 swing on it disappears into the
average. `imageStats` publishes exactly that mean plus variance, min, max and `litPixelFraction`;
`subjectRegion.subject.meanLuminance` is an average over the whole silhouette and moves ~0.003.
The pre-#2 response had no field that came back different. Current pose-set responses can publish
`poseSet.poseRepeatability` (see *Fix*), so this observation is historical and does not describe
current output.

## Mechanism, read from source

**Pre-#2 mechanism record (plugin commit `47307435`; line references and call counts below are
historical):** At that revision, `camera.orbit_shots` (`CameraFrameHandler.cpp:678`) built poses and handed them to the shared
primitive: `PinWrightPoseCapture::CaptureCameraPoses` -> `RunPoseListCapture`, which calls
`CaptureFrame` **once per pose** (`PoseListCapture.cpp:300`) plus one discarded warm-up
(`:181`). `CaptureFrame` calls `CaptureEditorViewportToPng`
(`PreviewViewportCaptureUtils.cpp:1643`). `camera.orbit_shots` binds no
`SubjectVisibilitySetter` and no coverage flag (grep: zero hits in `CameraFrameHandler.cpp`), and
the response carries no `subjectCoverage`, so there is exactly **one** capture per shot — 19
`CaptureEditorViewportToPng` calls for this 18-shot set.

For current pose-set callers such as `camera.orbit_shots`, `CaptureCameraPoses` owns one rig
around the set. Direct `CaptureEditorViewportToPng` callers may still create an owned
per-capture rig; the set-scoped claim below is specific to pose-set execution.

**Pre-fix source record (plugin commit `47307435`; superseded by #2 and #6):** At the revision
that produced the measured captures, this path did not drive or suppress temporal or
global-illumination accumulation. A grep of `PreviewViewportCaptureUtils.cpp/.h` for
`Lumen|DistanceField|DFAO|SkyLight|RealTimeReflection|Recapture` returned two comment hits and
no code (`PreviewViewportCaptureUtils.h:56`, `:1471`); `RecaptureSky` existed only in
`Handlers/Environment/LightingHandler.cpp:526,734`, off this path. Temporal AA was not
suppressed in that pre-fix path. This paragraph is historical and is not a current-source
claim.

**Current source facts:** one set-scoped viewport context arms fixed exposure, disables TemporalAA,
and creates the capture-resolution extension before it conditionally changes the viewport size
(`PreviewViewportCaptureUtils.cpp:674-704`); the capture path invokes that preparation before
applying the camera or pumping a frame (`PreviewViewportCaptureUtils.cpp:2086-2095`). Both capture
resolution fractions remain pinned by the view extension (`CaptureResolutionViewExtension.cpp:20-68`).
This closes the concrete unpinned pre-resize draw found in #8; this ticket still has no current
proof of a global-illumination accumulator as the cause.

**Pre-#2 performance record (plugin commit `47307435`; line references and pump counts below are
historical):** At that revision, each frame was pumped with `RequestRealTimeFrames(2)` + **3 unconditional pumps**
(`:2192-2196`), read (`:2218`), then the settle loop adds **at least one and at most eight** more
(`:2246-2280`). `PumpViewport` (`:107-125`) is Slate `PumpMessages` + `Tick` + two `Invalidate`s +
`SceneViewport->Draw()` + `FlushCommands()` + `FlushRenderingCommands()`. So the floor is four
draws per shot, and with `settleRounds:1` reported that is what this set paid.

**The original report identified two candidate carriers. Current source inspection confirmed an
unpinned synchronous draw before each per-shot resize; the current pose-set path removes both that
ordering gap and unchanged-size per-shot churn. Runtime confirmation remains pending because this
worker was prohibited from running the RHI-backed repeatability test.**

### 1. Preview-scene rig lifetime (pre-fix claim; corrected in current path)

**Pre-fix only (plugin commit `47307435`; this is not the current implementation):** At that
revision, `FScopedPreviewSceneRig` was constructed at exactly one site in the whole tree,
`PreviewViewportCaptureUtils.cpp:1906`, inside `CaptureEditorViewportToPng`, i.e. per shot. Its
constructor wrote the floor and environment visibility (`PreviewSceneRig.cpp:700-709`), and its
destructor restored the light state, shared profile array, and visibility
(`PreviewSceneRig.cpp:751-766,787-788`). On the 18-shot set, that old path wrote preview-scene
state 38 times. This is retained as historical diagnosis only.

The original mismatch was between those pre-fix mechanics and the set-level wire/docs contract:

- `CameraFrameHandler.cpp:706`, the `previewScene` parameter description an agent reads before
  calling: *"Read once for the whole set, so every shot in one call is lit identically, and
  restored once after the last one."*
- `CameraFrameHandler.cpp:774-777`: *"Applied and restored ONCE around the whole set by the capture
  primitive's scoped guard."*
- `CameraFrameHandler.cpp:1135-1136`: *"Set-level like the exposure pin: one rig applied and
  restored once around the whole orbit, **so no two shots in one set can be lit differently from
  each other or from their own report**."*

For pose-set callers such as `camera.orbit_shots`, those promises are now implemented by the
current set-scoped guard; they are not evidence that the current pose-set path rebuilds the rig.

**Pre-fix response evidence only:** the delivered response showed `viewport.previewScene` reported
once from the last capture with `showFloor: true` applied and **`previous.showFloor: false` and
`afterRestore.showFloor: false`**. That proved the old teardown; it does not describe current
behavior after #2.

For pose-set callers such as `camera.orbit_shots`, the current path constructs one rig and one
viewport-capture context around warm-up, requested poses, and, for non-time-driven sets, the final
control (`PoseListCapture.cpp:482-502`); individual captures only measure the rig and viewport pins
in force. Direct `CaptureEditorViewportToPng` callers create owned one-call scopes; the set-scoped
claim does not cover those direct calls. The old claim that the rig or viewport pins are repeatedly
written and reverted between shots is no longer current evidence or a current causal candidate.

### 2. Unpinned pre-resize draw and per-shot viewport churn (fixed in current path)

Positive `SetFixedViewportSize` calls route through `ResizeViewport` (`SceneViewport.cpp:1879-1889`),
whose resize path invalidates and calls `Draw()` synchronously when the client has a world
(`SceneViewport.cpp:1984-1995`). The current context now arms exposure, TemporalAA suppression,
and the resolution extension before its conditional positive resize
(`PreviewViewportCaptureUtils.cpp:674-704`). `CaptureCameraPoses` constructs that context once and
passes it through the set (`PoseListCapture.cpp:482-502`), so same-size shots do not resize again.
The context restores the original viewport size/fixed state before releasing those pins
(`PreviewViewportCaptureUtils.cpp:616-660`). This removes both the unpinned synchronous draw and
the established per-shot resize churn from the current path.

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

The earlier asks to publish pixel convergence, scope the preview-scene rig once per set, and add
same-pose `poseRepeatability` evidence are addressed in the current pose-set path and documented
in `Fix`. A stability gate remains deferred because no causal threshold has been established.

1. **Run one bounded exploratory causal A/B (future live verification; not run in this review).**
   Use a future test seam that selects only the viewport-sizing lifetime. Within the same session,
   hold the same request, static subject, settings, and pins fixed. Start each trial from a fresh,
   equivalent viewport/rig state, and use counterbalanced `ABBA` or `BAAB` trial order. In each arm,
   capture **8 consecutive identical-pose frames**:
   - Arm A: pre-fix/counterfactual per-shot `SetFixedViewportSize` sizing selected by the test seam.
   - Arm B: one set-scoped viewport-sizing lifetime for the whole pose set.
   Primary comparisons are adjacent-frame max pixel delta and the phase-4 swing in the same subject
   ROI. Compare published `poseSet.poseRepeatability` only as secondary context: it compares the
   first real pose-zero with the final discarded pose-zero control, not consecutive controls, so a
   period-4 cycle can alias and leave it unchanged. This experiment provides causal support only;
   no production code or test change is requested now.

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
  `CameraFrameHandler.cpp:706` as an existing "apply once for the whole set" seam. The statement
  that this seam "does not exist" was true only before #2; the current pose-set path provides the
  set-scoped seam and the docs match it. The remaining global-illumination question is deferred
  until the bounded viewport-sizing A/B rather than treated as a current cause.
- `B-capture-asset-preview-renders-foliage-black` (IN-REVIEW/High) — its `#1` names "whether the
  preview world has Lumen / DFAO / sky occlusion at all" as an untested candidate, then closes the
  candidate space on exposure. That global-illumination question remains unproven and is deferred
  until the bounded viewport-sizing A/B; no current attribution is made before that result.
- `B-mrq-artifact-report-omits-stream-count` `#2`, `B-ground-probe-hits-hull-not-render` (DONE/High),
  `B-thumbnail-cold-first-frame-no-stats` (DONE/High) — the recurring shape: the call succeeds,
  every reported number is correct, and the output is wrong because the number that mattered was
  never reported.

## Fix

**Verdict: TRUE.** The positive per-shot resize ran before exposure, TemporalAA suppression, and
the capture-resolution extension existed. UE 5.8 `SceneViewport.cpp:1988-1995` says "Invalidate,
then redraw immediately" and calls `Draw()` when the viewport client has a world, so each resize
could feed an unpinned frame into persistent render history before the requested capture state.

`CaptureCameraPoses` now owns one lazy set-scoped viewport context across warm-up, all real shots,
and the final control. It establishes all three pins before the first positive resize, skips the
resize while fixed state and dimensions already match, and restores the original size/fixed state
before releasing the pins. Direct `CaptureEditorViewportToPng` calls use a local one-call context.

Files changed: `Source/PinWright/Private/Handlers/Render/PreviewViewportCaptureUtils.{h,cpp}`,
`Source/PinWright/Private/Handlers/Render/PoseListCapture.{h,cpp}`,
`Source/PinWright/Private/Tests/Render/TestPoseListCaptureStability.cpp`,
`Source/PinWright/Private/Tests/Render/TestCaptureVerbParameterParity.cpp`, and
`Docs/wiki-src/render.md`. Test: `PinWright.render.pose_list.EightIdenticalPosesAdjacentMaxDelta`.

The historical `47307435` claim in #1 is unauditable because that object does not resolve in the
current plugin repository; it was not replaced with an unrelated hash. No engine source,
`CaptureResolutionViewExtension.*`, or `ErrorCodes.h` was changed. No build, editor capture, MCP
call, or automation run was performed under the worker restriction.

## History
- `#1-period-4-shading-cycle-reproduced` `OPEN` reporter — **Reproduced from the delivered pixels, not relayed.** `camera.orbit_shots` (verb established from the `CameraOrbit` filename prefix, written only at `CameraFrameHandler.cpp:1130-1131`), staticMesh `/Game/Atlantis/Meshes/SM_Column_Doric`, `previewScene {showFloor:true, showEnvironment:true}`, `exposure {mode:"fixed", ev100:-2.2869}`, 1120x1400, 18 shots in one call = 6 azimuths (144/145.5/147/148.5/150/151.5, el 8) x 3 repeats of the identical pose. Response `Saved/PinWright/HttpResponses/20260902T092641Z/20260902T171435Z_388389c9-…json`; stills `Saved/Screenshots/CameraOrbit/CameraOrbit_20260902_201429_shot00..17_*.png`, decoded with a stdlib PNG decoder whose whole-frame means reproduce the response's own `imageStats.meanLuminance` to 4 dp. FINDINGS: no same-pose repeat is identical — 65.5%-69.3% of the 1,568,000 pixels differ, mean abs 0.52-0.75, max 45-69/255 — against 13.08-13.29 mean abs for a real 1.5-degree move, so a repeat scores ~1/18 of a pose change but is not zero. The difference is sharply localised: a 28x28 patch at (504,1064), confirmed by cropping to be the base plinth's front face, reads 177.05/162.04/149.07/162.14/176.65/162.18/149.24/162.13/176.83/161.69/149.27/162.48/176.92/161.89/149.25/162.50/176.57/162.45 in capture order — **period exactly 4 by capture index, variance explained 1.000 (p2 0.015, p3 0.046, p5 0.027, p6 0.114, p7 0.238, p9 0.484), linear-trend R^2 vs index 0.000, phase sd 0.08-0.26 against a 27.98 amplitude**, i.e. deterministic; on that ROI the capture-index swing is 3x the azimuth effect (27.98 vs 9.36). NOT anti-aliasing jitter, on three tests: 84%-93% of >3/255 differences fall off high-gradient edges (edges are 0.53% of the frame), flat-interior signed means reach +0.25/-0.19 and flip sign with the phase, and the 8x-amplified signed diff shows filled faces and a diffuse ground darkening rather than silhouette rims — visually confirmed on a 3x zoom strip where the plinth face steps bright/mid/darkest/mid across three captures of one camera pose. A separate 1-LSB dither floor covers the whole frame (sky max diff exactly 1.0, signed mean 0.0006) and is plausibly `B-capture-render-resolution-unreported`'s effect; the two signals must not be collapsed. INVISIBLE TO EVERY PUBLISHED FIELD: `warmup {settled:true, settleRounds:1, meanLuminanceDelta:2.0808e-5}`, `exposure {pinned:true, adapted:4.880064, adaptedSource:"fixedPin"}`, `previewScene {applied:true, restored:true}`, `blank:false`, `litPixelFraction:1.0`, no warnings; the whole-frame mean moves ~0.001 normalised inside a triplet. MECHANISM (source, plugin HEAD `47307435`): one `CaptureEditorViewportToPng` (`PreviewViewportCaptureUtils.cpp:1643`) per pose (`PoseListCapture.cpp:300`) plus one warm-up (`:181`); `camera.orbit_shots` binds no coverage/visibility setter so there is no second draw per shot; nothing on the path waits for or suppresses Lumen/DFAO/skylight/temporal AA (grep of the capture utils for `Lumen|DistanceField|DFAO|SkyLight|RealTimeReflection|Recapture` yields two comments and no code); 3 unconditional pumps (`:2192-2196`) then a settle loop of 1-8 (`:2246-2280`). **Two established per-shot churns, neither proven to be the carrier:** (1) `FScopedPreviewSceneRig` is constructed at exactly one site tree-wide, `PreviewViewportCaptureUtils.cpp:1906`, i.e. per shot — ctor writes floor/environment visibility (`PreviewSceneRig.cpp:700-709`), dtor restores key-light direction/brightness/colour, sky brightness and floor/environment (`:751-766`, `:787-788`), so an 18-shot set writes preview-scene state 38 times; (2) `SetFixedViewportSize(W,H)` at `:1768` with `SetFixedViewportSize(0,0)` on scope exit at `:1731`, so the render target is unfixed and re-fixed between every pair of shots. **The set-level claim is false and it is on the wire:** `CameraFrameHandler.cpp:706` (the `previewScene` parameter description) says "Read once for the whole set, so every shot in one call is lit identically, and restored once after the last one", `:774-777` says "Applied and restored ONCE around the whole set", and `:1135-1136` says "one rig applied and restored once around the whole orbit, so no two shots in one set can be lit differently from each other or from their own report" — the exact invariant this ticket reports broken. Proven from the response alone: `viewport.previewScene` (reported from the last capture) has `showFloor: true` applied with `previous.showFloor: false` and `afterRestore.showFloor: false`, so the floor was off entering the last shot of a set that requested it throughout. A period of exactly 4 constrains any diagnosis: the carrying state survives across captures and advances per capture, so it is neither fully reset per shot nor converging. Severity High: silent wrong data on a normal path with every certifying field green, reach argued up because `CaptureEditorViewportToPng` is under every capture verb and the failing population is every asset-subject comparison set; not Critical (no crash, no asset write). Field cost RELAYED and not re-measured here: a 240-still turntable unusable, 34 steps above 3/255, largest 56/255.

- `#2-set-scoped-rig-and-repeatability-evidence` `IN-REVIEW` fixer — "Code inspection confirmed the per-shot rig lifetime and false set-level contract but not the period-4 cause. The real viewport wrapper now owns one rig across the complete set and patches the measured restore receipt after scope exit; non-time-driven sets add one discarded pose-0 control and publish pixel-difference evidence, while each capture's settle receipt carries the same local-change signals. Added pure/structural tests and updated the render, camera, and preview-rig wiki pages. Per-shot fixed-size viewport churn remains for a separate measured decision. No Unreal build, editor capture, or automation run was permitted in this worker pass."

- `#3-returned-viewport-churn-unproven` `OPEN` tester — "Returned by independent read-only verification: the rig is now scoped once per set, the wire docs match that lifetime, and the same-pose repeatability receipt landed, but `SetFixedViewportSize` is still applied and released per shot and no source evidence proves the period-4 shading cause. The named lighting-cycle defect therefore remains OPEN until a live capture can isolate the carrier or justify a set-scoped viewport lifetime."

- `#4-disposable-path-safety` `OPEN` developer — "Follow-up inspection found that the warm-up, coverage reference, and same-pose control were all deleted after capture but used deterministic caller-reachable names. They now request collision-checked generated names, and `PinWright.render.pose_list.PoseRepeatabilityEvidence` asserts all four discarded requests in its coverage-enabled fixture carry no caller filename. The period-4 lighting defect remains OPEN for the independent reason recorded in #3. No build, editor capture, or automation run was permitted."

- `#5-suite-3-contract-repair` `OPEN` developer — "Suite 3 exposed two follow-up contract gaps: the three set-wrapper fields added for rig lifetime and repeatability were not classified by the parameter-parity ratchet, and removing the fixed warm-up name also removed the primitive's non-empty warm-up filename. The fields are now explicitly classified as internal, and the warm-up uses a GUID-suffixed name so it remains reported to the capture primitive without risking overwrite-and-delete of a caller artifact. `PinWright.render.parameter_parity.PoseListSetFieldsAreReachableFromTheWire`, `PinWright.render.pose_list.WarmupFrameIsNotOneOfTheShots`, and `PinWright.render.pose_list.PoseRepeatabilityEvidence` cover the repair. The period-4 lighting defect remains OPEN. No build, editor capture, or automation run was permitted."

- `#6-current-code-causal-gap` `OPEN` developer — "Current-source evidence: `PoseListCapture.cpp:482-504` scopes one preview-scene rig across warm-up, requested poses, and final control; `PreviewViewportCaptureUtils.cpp:385-403` pins fixed exposure with `EyeAdaptation` disabled; `PreviewViewportCaptureUtils.cpp:1977-1995` suppresses temporal AA, while `CaptureResolutionViewExtension.cpp:20-68` pins primary and secondary screen-percentage fractions to 1.0. Per-shot viewport churn remains: `PreviewViewportCaptureUtils.cpp:1850,1892` clears and sets `SetFixedViewportSize`; UE routes positive sizing through `SceneViewport.cpp:1879-1900,1946-1982` (`ResizeViewport`/`UpdateViewportRHI`) and releases viewport resources at `SceneViewport.cpp:2128-2144` and `UnrealClient.cpp:2261-2265`. This evidence does not causally explain the four-shot cycle, so no speculative code or test was added."
- `#7-review-bounded-ab` `OPEN` developer — "Fix-review clarification: the pre-fix source/pixel diagnosis is historical. Current source scopes `FScopedPreviewSceneRig` once per set and disables `TemporalAA` for capture; positive viewport sizes route through `ResizeViewport`/`UpdateViewportRHI` and release viewport resources, while `SetFixedViewportSize(0,0)` only clears the fixed-size latch/invalidate and does not prove history rebuild. Replaced the broad Ask with one future causal A/B: same session/request/static subject/settings, 8 identical poses per arm, current per-shot sizing versus set-scoped sizing, compare phase-4 ROI swing and `poseSet.poseRepeatability`; no code or test change now. Status remains OPEN pending causal evidence."
- `#8-review-scope-and-aliasing` `OPEN` developer — "Review-fix clarification: pre-#2 hash/line/call-count and PumpMessages claims are historical to plugin commit `47307435`; current pose-set `camera.orbit_shots` execution uses one set-scoped rig while direct `CaptureEditorViewportToPng` callers may own a per-capture rig, and the current call is `EngineShowFlags.SetTemporalAA(false)`. Positive-size viewport/RHI churn is the only established/identified candidate; `(0,0)` does not prove history rebuild, and global-illumination attribution is deferred until the viewport A/B. The Ask and plan now specify an exploratory sizing-lifetime test seam, fresh equivalent viewport/rig state per trial, counterbalanced ABBA/BAAB order, 8 consecutive identical-pose captures per arm, primary adjacent-frame max pixel delta and phase-4 ROI swing, and secondary non-decisive `poseRepeatability` because it compares the first real pose-zero with the final discarded pose-zero control and can alias period 4. No code or test change was added; status remains OPEN."
- `#9-pre-resize-pin-gap` `IN-REVIEW` developer — "Set-scoped lazy capture context now owns exposure, TemporalAA suppression, and the capture-resolution extension before the first or changed-size viewport resize; unchanged pose-set shots do not resize, and direct single captures use a local one-call context. Historical 47307435 remains unauditable; live verification is pending."
