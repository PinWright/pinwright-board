---
id: E-sequencer-add-camera-track-no-transaction
title: "sequencer.add_camera_track records no undo transaction (bare Modify, no FScopedTransaction), so editor.undo cannot reverse it"
status: OPEN
severity: Low
category: ergonomic
tags: [mutator-no-undo-transaction, sequencer, add_camera_track, undo]
encounters: 1
lastSeen: 2026-07-02T03:00:21.2923847+03:00
---

# `sequencer.add_camera_track` mutates inside a bare `Modify()` with no `FScopedTransaction`, leaving `editor.undo` unable to reverse it

`sequencer.add_camera_track` adds a `UMovieSceneCameraCutTrack` to the MovieScene's
dedicated camera-cut slot inside a bare `MovieScene->Modify()` call with **no
enclosing `FScopedTransaction`** (`SequencerHandler.cpp` ~259-340). Because no
transaction is opened, the edit records no entry in the editor's undo buffer, so a
subsequent `editor.undo` has nothing to roll back — it cannot reverse an
`add_camera_track`.

On its own this is a robustness/ergonomic gap. What makes it worth tracking is that
it **compounds** `B-sequencer-remove-track-misses-camera-cut-slot` (OPEN): with
`sequencer.remove_track` structurally unable to touch the camera-cut slot,
`editor.undo` would be the natural typed-RPC fallback to reverse an
`add_camera_track` — but there is no undo entry to invoke. The net effect is that
there is **no typed-RPC path (neither `remove_track` nor `editor.undo`) to reverse
an `add_camera_track`**; the only recourse observed was a `python.execute` fallback
through UE's `MovieSceneSequenceExtensions.remove_track`.

The project already established the pattern of wrapping mutating MCP handlers in
`FScopedTransaction` (see `B-no-undo-redo`, DONE — widget, Blueprint, and BPIR
handler families were wrapped; the sequencer handler family was never in that
scope). Wrapping `add_camera_track` (and its sibling sequencer mutators —
`add_transform_track`, `add_animation_track`, etc.) in a scoped transaction would
restore `editor.undo` as a clean reversal path and bring the sequencer namespace in
line with the already-transactional widget/BP handlers.

## What it should do
Wrap `add_camera_track`'s (and sibling sequencer mutators') edit in an
`FScopedTransaction` (with the existing `Modify()` inside the scope) so a single
`editor.undo` reverses the authored track, matching the transactional convention
`B-no-undo-redo` established for the widget/Blueprint/BPIR handler families.

## Evidence (this task)
Source-inspection finding, surfaced by the struggle-audit of the
`sequencer.remove_track` cinematic task (`IntroEstablishingShot`, 18 calls) while
the agent hunted for a sanctioned reversal path after `remove_track` failed with
`[TRACK_NOT_FOUND]`. Reading the `add_camera_track` handler
(`SequencerHandler.cpp` ~259-340) the agent found the bare `MovieScene->Modify()`
with no `FScopedTransaction` and reasoned in SAY: "no undo entry was recorded —
editor.undo can't reverse it either." **The agent did not actually invoke
`editor.undo`** — this is a source-confirmed hunch, not an observed undo failure —
so it is filed Low, distinct from the primary `remove_track` defect the judge
already filed (`B-sequencer-remove-track-misses-camera-cut-slot`).

severity rationale: impact=pure-friction (no undo reversal path for the camera-cut
authoring verb; source-inspection hunch, `editor.undo` never invoked, and a
`python.execute` workaround exists) x reach=rare (sequencer camera-cut authoring) -> Low.

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
