---
id: E-sequencer-add-camera-track-no-transaction
title: "Most sequencer.* mutators (all of SequencerHandler.cpp, ~15 in SequenceHandler.cpp) open no FScopedTransaction, so editor.undo cannot reverse them"
status: OPEN
severity: Low
category: ergonomic
tags: [mutator-no-undo-transaction, sequencer, add_camera_track, undo]
encounters: 1
lastSeen: 2026-07-02T03:00:21.2923847+03:00
rice: [1, 1, 1, 3]
priority: 3
---

# Most sequencer mutators open no FScopedTransaction, so editor.undo cannot reverse them

Only some `sequencer.*` writers record an undo transaction. These open an `FScopedTransaction`:
`add_camera` (`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:803`),
`add_actor` / `add_actors` (`:975`, `:1110`), `add_keyframes` (`:2917`) and `remove_track`
(`:3745`), plus the Control Rig, bake and FBX-import verbs. The rest mutate the MovieScene with
a bare `Modify()` outside any transaction. Nothing is recorded in the undo buffer, so
`editor.undo` cannot reverse them. Under its contract it undoes whatever earlier transaction
is on top instead.

Untransacted mutators at 7230b41d:

- `Handlers/Sequencer/SequencerHandler.cpp`, which has no `FScopedTransaction` at all:
  `add_keyframe` (:63), `manage_track` (:245), `add_camera_track` (:345, `Modify` at :428),
  `add_camera_rig_rail` / `add_camera_rig_crane` (:563, :574), `add_level_visibility_track`
  (:585), `add_animation_track` (:693), `add_transform_track` (:774), `add_audio_track` (:840).
- `Handlers/Sequencer/SequenceHandler.cpp`: `set_display_rate` (:588), `set_properties`
  (:640), `add_spawnable_from_class` (:1447), `remove_actors` (:1527), `sequence.add_keyframe`
  (:2433), `add_section` (:3196), `set_tick_resolution` (:3308), `set_view_range` (:3392),
  `set_track_muted` / `set_track_solo` / `set_track_locked` (:3430, :3506, :3587), `add_track`
  (:3798), `add_sub_sequence` (:3989), `set_sub_section_range` (:4081), `set_work_range`
  (:4413).

Several of these have no inverse verb either. For example, nothing removes a section added by
`add_section` or `add_sub_sequence`, so `python.execute` is the only way back.

**Fix:** In each listed handler, after validation and before the first mutation, open an
`FScopedTransaction`. Call `Modify()` on the MovieScene and on any section or track before
writing to it. This is the convention `add_keyframes` documents at `SequenceHandler.cpp:2934`;
`add_camera_track` today calls `Modify()` only after `AddSection`/`SetRange` (:428). Cancel
the transaction on the error paths that run after a mutation.

**Acceptance:** For a representative set (`add_camera_track`, `add_transform_track`,
`add_section`, `add_sub_sequence`, `set_properties`), a test runs the verb then
`editor.undo`. `editor.undo` returns `undone[]` titled for that verb, and `list_tracks` /
`list_sections` / `get_properties` match the pre-call state. `editor.undo_history` shows one
entry per call.

**Workaround:** Reverse with the inverse verb where one exists (`remove_track`, which now
handles the camera-cut slot at `SequenceHandler.cpp:3724-3745`; `remove_actors`; re-setting the
property). Otherwise use `python.execute`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the
  `sequencer.remove_track` cinematic task (`IntroEstablishingShot`, 18 calls;
  outcome tool_bug, judge filed `B-sequencer-remove-track-misses-camera-cut-slot`).
  Distinct PROCESS surface, source-confirmed while working around the primary
  `remove_track` defect: `sequencer.add_camera_track` wraps its edit in a bare
  `MovieScene->Modify()` with no `FScopedTransaction` (`SequencerHandler.cpp`
  ~259-340), so no undo entry is recorded and `editor.undo` cannot reverse it —
  which compounds the `remove_track` camera-cut gap by removing the only other
  typed-RPC reversal path. Dedup: ripgrep across OPEN/DONE/WONTFIX — the existing
  undo tickets are BPIR-specific (`B-no-undo-redo` DONE covered widget/BP/BPIR
  handler families, never sequencer; `B-undo-last-bpir-doesnt-restore-phase0-sweeps`,
  `E-add-event-then-default-compile-bpir-unundoable` are BPIR-only); none covers a
  sequencer mutator lacking a transaction. New symptom-family tag
  `mutator-no-undo-transaction`. The agent did NOT invoke `editor.undo` (source
  hunch, not observed undo failure) -> filed Low. Fix: wrap `add_camera_track` and
  sibling sequencer mutators in `FScopedTransaction`, matching the pattern
  `B-no-undo-redo` established.
- `#2-rephrased` `OPEN` developer — Rescoped from `add_camera_track` alone to every untransacted sequencer mutator. `SequencerHandler.cpp` has zero `FScopedTransaction` across its 9 handlers, and 15 mutators in `SequenceHandler.cpp` use a bare `Modify()` (listed in the body with registration lines). Dropped the claim that this compounds `B-sequencer-remove-track-misses-camera-cut-slot`: `remove_track` now resolves and removes the camera-cut slot inside a transaction (`SequenceHandler.cpp:3724-3745`), so a typed reversal for `add_camera_track` exists. Fixed the stale `~259-340` citation (`add_camera_track` is registered at `SequencerHandler.cpp:345`, `Modify` at `:428`). Added the Modify-before-write requirement and Acceptance. Severity unchanged (Low: inverse verbs or Python cover reversal). RICE C 0.8->1 (verified in source), E 2->3 (rescope touches ~24 handlers across two files).
