---
id: B-sequencer-remove-track-misses-camera-cut-slot
title: "sequencer.remove_track cannot remove the camera-cut track (returns TRACK_NOT_FOUND for a track list_tracks surfaces); never scans GetCameraCutTrack()"
status: IN-REVIEW
severity: Medium
category: bug
tags: [camera-cut-slot-not-mutable, sequencer, remove_track, camera-cut, mutator]
encounters: 1
lastSeen: 2026-07-02T02:58:35.5974840+03:00
---

# `sequencer.remove_track` cannot remove a camera-cut track it can create and list

`sequencer.add_camera_track` creates a `UMovieSceneCameraCutTrack` on the
MovieScene's **dedicated** camera-cut slot (via `MovieScene->AddCameraCutTrack(...)`,
`SequenceHandler.cpp` add_camera_track handler / `SequencerHandler.cpp:295`), and
`sequencer.list_tracks` now enumerates that slot and reports it as a real track
(`trackName: "MovieSceneCameraCutTrack_0"`, `isCameraCutTrack: true`,
`SequenceHandler.cpp:2840-2845`). But `sequencer.remove_track` scans **only**
`MovieScene->GetTracks()` (master tracks) and each `Binding.GetTracks()`
(per-binding tracks) — it **never** consults `MovieScene->GetCameraCutTrack()`.
The camera-cut track does not live in either of those collections (it is a
separate UPROPERTY slot), so `remove_track` walks off the end and returns
`[TRACK_NOT_FOUND] Track not found` — for a track that demonstrably exists and
that its own sibling readers (`list_tracks`, `get_camera_cut_track`,
`list_sections`) all surface.

The result is a misleading error and a self-inconsistency: the plugin can
**create** a camera-cut track (`add_camera_track`) and **enumerate** it
(`list_tracks`), but cannot **remove** it through the obvious mutator. The
`trackName` string that `list_tracks` hands back is rejected by `remove_track`
that is meant to consume it. There is no `sequencer.*` RPC that removes a
camera-cut track; the only recourse is `python.execute` calling UE's own
`MovieSceneSequenceExtensions.remove_track`, which special-cases the cut class to
call `RemoveCameraCutTrack()`.

Note the deeper reason the scan can't just be widened blindly: even if the loop
matched the cut track, `MovieScene->RemoveTrack(*Track)` operates on the `Tracks`
array and would not clear the dedicated camera-cut slot — removing it requires
the engine's `MovieScene->RemoveCameraCutTrack()` path (what the Python
extension special-cases). So the fix is a real branch, not a one-line loop tweak.

## What it should do
`remove_track`, when the requested `trackName` matches
`MovieScene->GetCameraCutTrack()` (by `GetName()`/`GetDisplayName()`, same
matching it already uses for master tracks), should call
`MovieScene->RemoveCameraCutTrack()` (the versioned equivalent) and report
success — so a track `list_tracks` surfaces is removable by the name it reports.
Removing a camera-cut track through `add_camera_track`'s sibling mutator should
round-trip; TRACK_NOT_FOUND should never be returned for a track the plugin's
own readers enumerate.

## Verbatim repro (replay-confirmed live at HEAD)
1. `sequencer.create {name: RemoveTrackReplay, path: /Game/Cinematics}` -> ok, `/Game/Cinematics/RemoveTrackReplay`
2. `sequencer.add_camera {path: /Game/Cinematics/RemoveTrackReplay}` -> ok, `cameraActorPath: .../PersistentLevel.CameraActor_7`
3. `sequencer.add_camera_track {sequencePath: /Game/Cinematics/RemoveTrackReplay, cameraActorPath: .../CameraActor_7, startTime:0, endTime:5}` -> `success:true`
4. `sequencer.list_tracks {path: /Game/Cinematics/RemoveTrackReplay}` -> `{"tracks":[{"trackName":"MovieSceneCameraCutTrack_0","trackType":"MovieSceneCameraCutTrack","displayName":"Camera Cuts","isMasterTrack":true,"sectionCount":1,"isCameraCutTrack":true}],"trackCount":1}`
5. `sequencer.remove_track {path: /Game/Cinematics/RemoveTrackReplay, trackName: "MovieSceneCameraCutTrack_0"}` -> **`[TRACK_NOT_FOUND] Track not found`**

The `trackName` in step 5 is copied verbatim from step 4's `list_tracks` output.
The attempt agent also tried the `displayName` ("Camera Cuts") and the bare class
name ("MovieSceneCameraCutTrack") — all three return TRACK_NOT_FOUND, since the
scan never reaches the slot regardless of the name variant.

## Guilty source (verbatim)
`Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`, `sequencer.remove_track` handler (registered `:2369`):

```cpp
    for (UMovieSceneTrack* Track : MovieScene->GetTracks())            // :2397 master tracks only
    {
        if (Track && Track->GetName().Contains(TrackName))
        {
            RemovedTrackName = Track->GetName();
            MovieScene->RemoveTrack(*Track);
            bRemoved = true;
            break;
        }
    }

    if (!bRemoved)
    {
        for (const FMovieSceneBinding& Binding :                        // :2410 per-binding tracks only
             const_cast<const UMovieScene*>(MovieScene)->GetBindings())
        {
            for (UMovieSceneTrack* Track : Binding.GetTracks())
            { ... MovieScene->RemoveTrack(*Track); ... }
        }
    }
    ...
        Ctx.SendError("TRACK_NOT_FOUND", "Track not found");            // :2438
```

No branch consults `MovieScene->GetCameraCutTrack()` — the exact slot that the
same file's `sequencer.list_tracks` enumerates at `:2840-2845`
(`if (UMovieSceneTrack* CameraCutTrack = MovieScene->GetCameraCutTrack()) { ... isCameraCutTrack ... }`).

severity rationale: impact=blocker-with-python-workaround (can't remove via any sequencer.* RPC; misleading TRACK_NOT_FOUND on a track list_tracks/get_camera_cut_track both surface) x reach=rare (sequencer camera-cut-track removal) -> Medium (the misleading error on an enumerable track + create-but-can't-remove asymmetry keeps it above Low).

## History
- `#2-fix-camera-cut-removal` `IN-REVIEW` developer — Added a camera-cut branch to the `sequencer.remove_track` handler (`Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`, after the GetTracks()/binding loops, before the TRACK_NOT_FOUND SendError): when the requested `trackName` matches `MovieScene->GetCameraCutTrack()` by `GetName()` OR `GetDisplayName()` (both fields `list_tracks` reports at `:2818/:2820`), it clears the dedicated slot via the engine's `MovieScene->RemoveCameraCutTrack()` (`MovieScene.h:757`; nulls the `CameraCutTrack` UPROPERTY — verified `MovieScene.cpp:1370`) and reports success. A bare `RemoveTrack()` (Tracks-array only) would not clear the slot, so this is a real branch not a loop widen — matches the ticket's Fix. The camera-cut trackName `list_tracks` hands back now round-trips through `remove_track`. Regression test `PinWright.Sequencer.RemoveTrack.RemovesCameraCutTrack` (`Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestRemoveTrackCameraCut.cpp`, sibling of the list-side `TestListTracksCameraCut.cpp`): drives the real handler against an in-code transient sequence with a camera-cut track authored via `MovieScene->AddCameraCutTrack(...)`; asserts (1) a bogus name still returns `TRACK_NOT_FOUND` and leaves the slot intact, (2) removal by the reported `GetName()` succeeds and `GetCameraCutTrack()` becomes null, (3) removal by the `displayName` also clears the slot. Reverting the branch makes case (2)/(3) fail with TRACK_NOT_FOUND. Uses no example-content assets (fixture built in-code). Files: `SequenceHandler.cpp`, `Tests/Sequencer/TestRemoveTrackCameraCut.cpp`.
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live at HEAD. SEED-mode `sequencer.remove_track` cinematic task: created `/Game/Cinematics/RemoveTrackReplay`, `add_camera`, `add_camera_track`, `list_tracks` reported the cut track as `trackName:"MovieSceneCameraCutTrack_0" isCameraCutTrack:true`, then `remove_track {trackName:"MovieSceneCameraCutTrack_0"}` -> `[TRACK_NOT_FOUND]`. Root: the `remove_track` handler (`SequenceHandler.cpp:2369`) scans only `MovieScene->GetTracks()` (`:2397`) and per-binding `Binding.GetTracks()` (`:2410`), never `MovieScene->GetCameraCutTrack()` — the dedicated slot `add_camera_track` writes and `list_tracks` (`:2840-2845`) now reads. Fix needs a real branch calling `MovieScene->RemoveCameraCutTrack()` (bare `RemoveTrack` on the cut track won't clear the slot). Dedup: ripgrep OPEN/DONE/WONTFIX — `E-sequencer-list-tracks-omits-camera-cut-undocumented` (IN-REVIEW) covers the LIST side (fixed to surface the cut track) and `F-rpc-sequencer-get-camera-cut-track` (DONE) the READ side; neither covers removal. New symptom-family `camera-cut-slot-not-mutable`: the dedicated GetCameraCutTrack() slot is readable/enumerable but the mutator never scans it. Distinct from `F-niagara-remove-emitter` (different subsystem).
