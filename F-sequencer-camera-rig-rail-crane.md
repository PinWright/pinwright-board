---
id: F-sequencer-camera-rig-rail-crane
title: "Add typed handlers for Camera Rig Rail and Crane in Sequencer"
status: DONE
severity: Medium
category: feature
tags: [sequencer, camera-rig, ergonomic]
---

# Add typed handlers for Camera Rig Rail and Crane in Sequencer

CoPilot-class workflows that author cinematic camera moves expect first-class `sequencer.add_camera_rig_rail` and `sequencer.add_camera_rig_crane` handlers, mirroring the existing typed convenience for camera-cut tracks (`sequencer.add_camera_track` in `SequencerHandler.cpp`). Each new handler should:

1. Spawn the rig actor (`ACameraRigRail` / `ACameraRigCrane`) into the sequence — or bind to an existing one passed by actor path — and add a possessable/spawnable binding.
2. Add the corresponding track to that binding (`UMovieSceneCameraRigRailTrack` / `UMovieSceneCameraRigCraneTrack`) with a default section spanning the playback range.
3. Return the binding GUID, spawned/bound actor path, and a `track` payload using the shared `BuildTrackJson` helper from `Utils/MovieSceneJsonUtils.h` so the response is byte-identical to dump/`list_tracks` output.

Today, callers have to chain `sequencer.add_spawnable_from_class` (with `/Script/CinematicCamera.CameraRigRail` or `.CameraRigCrane`), then `sequencer.add_track` with the raw track class string discovered via `list_track_types`, then re-query bindings to find the GUID. The rail path is even worse: `ACameraRigRail` carries a spline component, and there's no convenience for inspecting/initialising spline points — naive `add_spawnable_from_class` produces a zero-length rail that's invisible until the user manually keys spline points in the editor.

Typed handlers unblock keyframing of the rig-specific channels (`CurrentPositionOnRail` on rail, `CraneYaw`/`CranePitch`/`CraneArmLength` on crane) via existing `sequencer.add_key` / `sequencer.set_keys` flows, since the binding GUID returned here can be fed straight into those calls.

**Fix:** Add two handlers next to `add_camera_track` in `SequencerHandler.cpp`:

- `sequencer.add_camera_rig_rail` — required `sequencePath`; optional `actorPath` (possess existing `ACameraRig_Rail`) or default to spawning the rig as a spawnable. Adds a `UMovieScene3DTransformTrack` plus a `UMovieSceneFloatTrack` keyed to the rig's `CurrentPositionOnRail` property. Returns `{ sequencePath, bindingGuid, actorPath, tracks: [...] }` with each entry shaped by the shared `BuildTrackJson` helper.
- `sequencer.add_camera_rig_crane` — same shape with `ACameraRig_Crane`, adding the transform track plus float property tracks on `CranePitch`/`CraneYaw`/`CraneArmLength`.

No dedicated `UMovieSceneCameraRig*Track` classes exist in UE 5.6 — the engine animates these rigs via generic property tracks on `Interp`-tagged float fields, so the typed handlers here scaffold the standard binding + transform + property-track shape callers would otherwise chain together by hand. `Build.cs` adds `"CinematicCamera"` to public dependencies (engine module that owns `ACameraRig_Rail`/`ACameraRig_Crane`).

Optional follow-up: `sequencer.set_rail_spline_points` to author the rail's spline in one call — out of scope, file separately if needed.

## History
- `#1-initial-repro` `OPEN` reporter — No existing handler for camera rig rail/crane: grep of `camera_rig`/`CameraRigRail`/`CameraRigCrane`/`MovieSceneCameraRig` across `docs/rpc-method-reference.generated.md` and `Private/Handlers/Sequencer/` returned zero matches. Current path requires `add_spawnable_from_class` + raw `add_track` chain plus manual GUID lookup; rail spline initialisation has no convenience either. `F-rpc-sequencer-get-camera-cut-track` (DONE) and `F-rpc-sequencer-list-sections` (DONE) already establish the `BuildTrackJson` / `MovieSceneJsonUtils.h` helper pattern this would reuse.
- `#2-reshape-and-implement` `IN-REVIEW` developer — Reshaped Fix: ticket-named track classes `UMovieSceneCameraRigRailTrack`/`UMovieSceneCameraRigCraneTrack` do not exist in UE 5.6; engine animates `ACameraRig_Rail`/`ACameraRig_Crane` (underscored, module `CinematicCamera`) via generic property tracks. Implemented `sequencer.add_camera_rig_rail` and `sequencer.add_camera_rig_crane` in `SequencerHandler.cpp` that spawn or possess the rig actor, add a `UMovieScene3DTransformTrack`, and add `UMovieSceneFloatTrack`s on the rig-specific `Interp` properties (`CurrentPositionOnRail` for rail; `CranePitch`/`CraneYaw`/`CraneArmLength` for crane). Added `CinematicCamera` to `Build.cs` public deps. Appended four no-crash dispatcher tests to `Tests/Media/TestSequencerHandlers.cpp`.
- `#3-verify-fix` `DONE` tester — Verified: both handlers wired and execute end-to-end against `/Game/NewLevelSequence`. `sequencer.add_camera_rig_rail` returned `{mode:"spawned", bindingGuid, actorPath:"/Script/CinematicCamera.CameraRig_Rail", tracks:[Transform(MovieScene3DTransformTrack), CurrentPositionOnRail(MovieSceneFloatTrack)]}`; `sequencer.add_camera_rig_crane` returned the same shape with `tracks:[Transform, CranePitch, CraneYaw, CraneArmLength]` — all float tracks shaped via `BuildTrackJson`. Schema discovery via `sequencer.add_camera_rig_rail?` exposes the documented `sequencePath` (required) + `actorPath` (optional) params.
