---
id: F-animated-capture-verbs
title: "No frame-burst capture for animated skinned meshes; camera.animation_shots and render.capture_animation_preview"
status: DONE
severity: Medium
category: feature
tags: [camera, render, sequencer, persona, skeletal-mesh, animation, visual-verification]
---

# Frame-burst capture verbs for animated skinned meshes

Reviewing a moving silhouette was only possible by hand-rolling a scrub-then-capture loop across
four verbs (`actor.spawn`, `sequencer.create`/`add_actor`/`add_animation_track`,
`sequencer.set_playhead`, `camera.orbit_shots`), and nothing about that loop was written down. So
it was skipped, and a skinned creep layer shipped in bind pose.

Shipped two verbs returning the same per-shot fields, so a review moves between them unchanged:

- **`camera.animation_shots`** — a placed actor in the level, driven by scrubbing a Level Sequence
  playhead. N instants crossed with M camera angles in one call.
- **`render.capture_animation_preview`** — an asset in isolation, driven by the Persona preview
  component. No level, no placed actor, no sequence.

## The finding that would have silently ruined every multi-angle set

`FEditorViewportClient::CalcViewRotationMatrix` (`EditorViewportClient.cpp:7392-7406`) **ignores
`InViewRotation` entirely** when `bUsingOrbitCamera` and derives the view from `ComputeOrbitMatrix()`
instead. A Persona viewport runs with orbit **on** by default —
`FAnimationViewportClient::SetCameraFollowMode` sets `bUsingOrbitCamera = Mode != BoneAsCamera`
(`AnimationEditorViewportClient.cpp:406`) and the default mode is `None`. An unfixed port of the
static-mesh capture path would have thrown away every requested azimuth/elevation and returned a
six-side set shot six times from one angle, with **nothing in the response to say so**.
`render.capture_asset_preview` is unaffected: `bUsingOrbitCamera` defaults false and the Static Mesh
editor never enables it.

## Numeric proof the pose changed, so a capture set is not the only evidence

Component-space bone transforms are sampled at every instant and compared against **both** the
previous instant and the first (neighbour-only misses a pose that moves and returns; first-only
misses a slow drift). Component space is deliberate: it measures the **pose** and nothing else, so
an actor travelling across the level moves no bone. That makes pose change and actor motion
independent, and their combination a diagnosis:

| `poseChanged` | `actorTranslationCm` | Verdict |
|---|---|---|
| false | > 0 | **sliding in bind pose** — a transform track with no skeletal animation track behind it |
| false | ~0 | nothing moved; comparing the images proves nothing |
| — | — | `poseSampled: false` means the animation system never evaluated; the images cannot be trusted |

## Runtime verification (integration pass 4)

- **Orbit suppression works.** `views:"sides"` on `SKM_Manny` + `MM_Walk_Fwd` returned six shots
  with six distinct camera rotations, locations, file sizes (55547 / 386655 / 378863 / 42493 /
  508792 / 38596 B) and mean luminances. **Caveat:** `orbitCameraDisabled` reported `false`, so the
  flag itself is not independent evidence — there is no public getter for `bUsingOrbitCamera` and
  it is inferred from whether the stored pose changed. The outcome is right; the flag is unproven.
- **Explicit pose evaluation lands.** `SetPlaying(false)` → `SetPosition` → `TickAnimation(0.f)` →
  `RefreshBoneTransforms(nullptr)` moved 156 of 161 bones at every instant, up to 60.7 cm and
  63.3 degrees. The `GlobalAnimRateScale`-zeroed fallback is not needed.
- **24 shots = 24 viewport resize cycles is safe.** Two back-to-back 24-shot bursts (48 cycles, on
  top of ~48 earlier in the same session) against the `FViewport::GetHitProxy` assert class that
  cost 66 creeps: editor alive and responding, **0** log matches for
  `GetHitProxy|Assertion failed|Fatal error`. The `SetFixedViewportSize` fallback is not needed.
- **In-level burst works end to end.** Four instants of a walk cycle rendered as four visibly
  different poses with four distinct file sizes, `poseChanged: true`, `restoredPlayhead: true`,
  `blankShots: 0`, one capture size (640) held for the whole burst.
- **A false alarm worth knowing about.** The first burst returned 24 shots with only 6 distinct
  file sizes, byte-identical across all four instants — exactly the "not animating" signature. It
  was fixture placement, not the verb: the review actor was outside the loaded world-partition
  region (frames contained only the editor grid), and then buried inside terrain (camera inside
  geometry, photographing gravel). Seating it on the landscape produced distinct frames. Note
  `poseChanged` reported `true` in all three placements, correctly — it measures bone transforms,
  not pixels. **When a burst looks frozen, check the actor is visible before suspecting the verb.**

## Extractions

`SequencePlayheadUtils.h` (out of `sequencer.set_playhead`) and `CameraShotPlanUtils.h` (out of
`camera.frame_actor`), both reusing the original namespace names so no call site changed. A second
scrub implementation is how one of them keeps a stale engine-version branch or drops the forced
re-evaluation and silently records an off-by-one frame across a whole burst.
`PinWright.Sequencer.SetPlayhead.*` (5 tests) still passes after the extraction.

## History
- `#1-designed-and-built` `IN-REVIEW` developer — New: `Handlers/Render/AnimationShotsHandler.cpp`, `AnimationPreviewCaptureHandler.cpp`, `AnimationPoseEvidence.h`, `CameraShotPlanUtils.h`, `Handlers/Sequencer/SequencePlayheadUtils.h`, `Tests/Render/TestAnimationCaptureHandlers.cpp` (13 tests). Edited `CameraFrameHandler.cpp` (-235 lines), `SequenceHandler.cpp`, `RenderHandler.cpp` (rejection message now names the new verb). No `ErrorCodes.h` and no `Build.cs` change. Not built or run at authoring time.
- `#2-runtime-verified` `DONE` tester — Built (UE 5.8, `Result: Succeeded`, zero errors and zero warnings in both logs, fresh DLL timestamps — not the exit code). Both extracted headers landed in the SAME unity TU as their consumers with `-DisableAdaptiveUnity` (`Module.PinWright.17.cpp` merged 7 touched TUs) and did not collide, which was the ODR risk. All 13 tests pass, `SetPlayhead.*` regression suite intact, and the four runtime checks above were exercised live on `/Game/Maps/Dota2_Blockout`. Committed as `9da255f6`.
