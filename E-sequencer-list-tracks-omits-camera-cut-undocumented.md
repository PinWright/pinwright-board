---
id: E-sequencer-list-tracks-omits-camera-cut-undocumented
title: "sequencer.list_tracks silently omits the camera-cut/root track and the wiki never warns; a 'list the tracks' intent misses it"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, sequencer, list_tracks, camera-cut, root-track, discovery, readback]
---

# `sequencer.list_tracks` silently omits the camera-cut track and the wiki never warns of it

`sequencer.list_tracks` enumerates only `MovieScene->GetTracks()` and the
per-binding `Binding.GetTracks()` (handler at `SequenceHandler.cpp:2781`; the
enumeration loops are at `SequenceHandler.cpp:2828-2855`). The camera-cut
track lives on a separate dedicated slot (`MovieScene->GetCameraCutTrack()`), so
it is **not** included in the `tracks` array. A caller whose intent is the
generic "list the tracks on this sequence and report what's there" reaches for
`list_tracks` — the obvious method for that intent — and silently gets an
incomplete answer: the camera-cut track they just authored via
`sequencer.add_camera_track` is missing, with no field, count, or note signalling
the omission. They only discover the gap if they happen to know to also call
`sequencer.get_camera_cut_track`.

This is the **process/docs** companion to `F-rpc-sequencer-get-camera-cut-track`
(DONE), which added the dedicated `sequencer.get_camera_cut_track` reader so the
camera-cut track is *readable somewhere*. That ticket's fix shape (1) was the
dedicated reader; its shape (2) — "extend `sequencer.list_tracks` to include the
camera-cut track in its `tracks` array (with a discriminator like
`isCameraCutTrack: true`)" — was explicitly left as a *"consider as a
follow-up"* and never landed or filed. So today `list_tracks` still omits it,
and nothing tells the caller.

The `docs/wiki-src/sequencer.md` overlay makes this worse by omission: the "Live
readers and dump parity" section documents that **`list_sections`** emits
camera-cut sections and that **`get_camera_cut_track`** exists, but says nothing
about **`list_tracks`** at all — there is no `list_tracks` entry and no warning
that it omits the camera-cut/root track. A caller reading that section
reasonably infers the readers are complete and uses `list_tracks` for the
track-enumeration intent.

## Evidence (this task)

SEED-mode `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls).
The run added a camera-cut track (`sequencer.add_camera_track`), then at the
"list the tracks on the sequence and report what's there" step called
`sequencer.list_tracks` — which returned only the transform track, omitting the
camera-cut track — and had to fall to `sequencer.get_camera_cut_track` plus
`sequencer.list_sections` to confirm the camera-cut track existed. Verbatim
friction note: "sequencer.list_tracks omits the camera-cut (root/master) track
entirely - had to use get_camera_cut_track/list_sections to confirm it." All
calls `ok=true`; the omission is silent — `list_tracks` reports success with an
incomplete list, and the gap surfaces only when the caller separately probes the
dedicated reader.

## What to do

Two complementary mitigations; the doc note is cheap and lands regardless of
whether the code extension is taken:

1. **Wiki note (this ticket's primary, downstream wiki process).** Add a short
   `sequencer.list_tracks` note to `docs/wiki-src/sequencer.md` (and mention it
   in the "Live readers and dump parity" bullet list) stating that `list_tracks`
   enumerates root/master and per-binding tracks **but not the camera-cut/root
   track**, and that the camera-cut track is read via
   `sequencer.get_camera_cut_track` (full object) or `sequencer.list_sections`
   (its sections). This sets the expectation up front so a "list the tracks"
   intent budgets the extra `get_camera_cut_track` call instead of discovering
   the omission by readback.
2. **Optional code follow-up (re-opens `F-rpc-sequencer-get-camera-cut-track`
   shape 2).** Extend `list_tracks` to include the camera-cut track in `tracks`
   with an `isCameraCutTrack: true` discriminator, so generic track enumerators
   no longer miss it. If this lands, tighten/remove the wiki note.

**Workaround (today):** after authoring a camera-cut track, do not rely on
`sequencer.list_tracks` for a complete picture — also call
`sequencer.get_camera_cut_track` (or read camera-cut sections via
`sequencer.list_sections`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the SEED-mode
  `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls; the
  per-finding judge filed the section-range gap
  `F-sequencer-transform-section-range-not-expanded-by-keyframe`). This ticket
  covers a distinct PROCESS surface from the friction note:
  `sequencer.list_tracks` returned only the transform track, omitting the
  camera-cut/root track, forcing `get_camera_cut_track` + `list_sections`
  readbacks to confirm it. Root: `list_tracks` (handler `SequenceHandler.cpp:2781`,
  loops `:2828-2855`) iterates only `MovieScene->GetTracks()` + per-binding tracks; the camera-cut
  track is on the separate `GetCameraCutTrack()` slot (same root that
  `F-rpc-sequencer-get-camera-cut-track` #1 established). Dedup: ripgrep across
  OPEN/DONE/WONTFIX — `F-rpc-sequencer-get-camera-cut-track` (DONE) fixed the
  *readability* via a dedicated reader but explicitly deferred the `list_tracks`
  extension as a "consider as a follow-up" that was never filed, and no ticket
  tracks the `list_tracks` omission or the missing wiki warning; the wiki
  "Live readers and dump parity" section documents `list_sections` /
  `get_camera_cut_track` but never `list_tracks`. Sibling to
  `E-sequencer-set-track-state-no-readback-doc` (docs companion to a sequencer
  read-gap). Primary ask: one-line `list_tracks` warning in
  `docs/wiki-src/sequencer.md`; optional code follow-up re-opens
  `F-rpc-sequencer-get-camera-cut-track` shape 2.
- `#2-additional-list-tracks-camera-cut-omission` `OPEN` reporter — Additional
  evidence (independent task, `sequencer.add_actor` SEED): built a `UELogoShowcase`
  cinematic — possessed UELogo + UELogo2 via `sequencer.add_actor`, spawned a
  camera, added a transform track on UELogo, then `sequencer.add_camera_track`
  (`{sequencePath, cameraActorPath, startTime:0, endTime:5}` → `success:true`).
  REPLAY-CONFIRMED the omission live: `sequencer.list_tracks {path:
  /Game/Cinematics/UELogoShowcase}` returned **`trackCount: 2`** with only the two
  `MovieScene3DTransformTrack` entries — no camera-cut entry, no field/count/note
  signalling it — while `sequencer.get_camera_cut_track` on the same sequence
  returned a live `{class:"MovieSceneCameraCutTrack", sectionCount:2, sections:[…]}`.
  Same silent-omission behavior as #1, reproduced on a fresh sequence via a
  different authoring path; confirms the gap persists. No code change since #1.
- `#3-retriage` `OPEN` triage — Low→Medium: list_tracks silently omits the camera-cut track with no signal, a silent-incomplete readback that misses an authored track on a niche sequencer path.
- `#4-fix` `IN-REVIEW` developer — Reword + fix. Reword: corrected the stale `SequenceHandler.cpp:2379` cite (that line is inside `sequencer.remove_track`, not `list_tracks`) to the real handler at `:2781` with the enumeration loops at `:2828-2855`, in the body and History #1. Fix (both mitigations): (1) code — extended the `sequencer.list_tracks` handler (`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`, after the master-track loop at ~:2828) to also emit the camera-cut track from the dedicated `MovieScene->GetCameraCutTrack()` slot, tagged `isCameraCutTrack: true` (+ `isMasterTrack: true`), reusing the same `EmitTrack` lambda so a generic "list the tracks" intent no longer silently drops an authored cut track; this lands the deferred shape (2) of `F-rpc-sequencer-get-camera-cut-track`. (2) wiki — added a `sequencer.list_tracks` bullet to the "Live readers and dump parity" section of `docs/wiki-src/sequencer.md` documenting that it enumerates root/master + per-binding + the camera-cut track (the `isCameraCutTrack` entry), with `get_camera_cut_track`/`list_sections` for the full object/sections. Regression test: `Source/PinWright/Private/Tests/Sequencer/TestListTracksCameraCut.cpp` (`PinWright.Sequencer.ListTracks.IncludesCameraCutTrack`) authors a camera-cut track via the engine `AddCameraCutTrack` slot (asserting it is absent from `GetTracks()`), drives the real `list_tracks` handler, and asserts exactly one `isCameraCutTrack:true` entry appears (zero before authoring); reverting the handler emission fails it.
