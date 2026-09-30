---
id: B-sequencer-empty-trackname-matches-first
title: "An omitted trackName silently targets the first track in five sequencer verbs"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, footgun, silent-wrong-target]
encounters: 1
---

# An omitted trackName silently targets the first track

`trackName` is `RPC_PARAM_OPT` on `sequencer.set_track_muted`, `set_track_solo`,
`set_track_locked`, `remove_track` and `add_section`, and every one of them resolves it with
`FString::Contains`. `FString::Contains("")` is **true**, so a call that omits `trackName`
matches the *first* track it walks and mutates that one — muting, soloing, locking or deleting an
arbitrary track the caller never named.

This is not in the same tier as the fake-success class: the response reports the real name of the
track it actually touched, so a caller who reads the response can tell. But `remove_track` with no
`trackName` deletes a track, and the read that would reveal it is one an agent has no reason to
make after a success. Verified at `b92ba268` in
`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` — after the
`B-sequencer-add-track-returns-unresolvable-name` fix the predicate is a single shared helper,
`SequenceHelpers::TrackMatchesIdentifier`, which preserves the old semantics deliberately so the
behaviour change could be considered separately.

**Fix:** either reject an empty/absent `trackName` with `INVALID_ARGUMENT` on the mutating verbs
(`add_section`'s no-name case is arguably a convenience worth keeping and can stay), or make
`TrackMatchesIdentifier` return false for an empty query and let the existing `TRACK_NOT_FOUND`
path fire. The second is a one-line change now that the predicate is shared, but it changes
behaviour for any caller relying on the implicit first-track default, so it wants its own test
pass.

## History
- `#1-found-during-identifier-unification` `OPEN` reporter — Found at `b92ba268` while unifying the
  sequencer track-identifier emit/resolve sites for
  `B-sequencer-add-track-returns-unresolvable-name`. Deliberately left unchanged in that fix so the
  identifier work stayed behaviour-preserving on the lookup side. Repro: call
  `sequencer.set_track_muted {path:"<seq with two or more tracks>", muted:true}` with no
  `trackName` — the first track is muted and its name is reported back.
- `#2-re-rated` `OPEN` triage — Severity Medium -> High. Impact class is Critical for `remove_track`: omitting the optional `trackName` deletes whichever track is walked first, and the handler (`SequenceHandler.cpp` `sequencer.remove_track`) opens no `FScopedTransaction`, so the lost track cannot be undone; reach bumps it down one because calling a mutating verb without `trackName` is an edge path, and the response does name the track it removed.
- `#3-empty-name-matches-nothing` `IN-REVIEW` developer — Changed
  `SequenceHelpers::TrackMatchesIdentifier` in
  `Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` to return false for an empty
  query, so every caller (`add_section`, `set_track_muted`, `set_track_solo`, `set_track_locked`,
  `remove_track`; `list_sections` already skipped an empty filter) answers `TRACK_NOT_FOUND` and
  touches nothing. `trackName` on those five verbs is now `RPC_PARAM_REQ`, so an absent name is
  rejected by the dispatcher with `MISSING_REQUIRED_PARAM` before the handler runs. `add_section`'s
  no-name convenience was dropped too: it picked an arbitrary track the same way. `remove_track` now
  resolves the track first and removes it inside one `FScopedTransaction` ("Remove Sequencer
  Track"); `UMovieScene::RemoveTrack` / `RemoveCameraCutTrack` call `Modify()` themselves. Documented
  under `### sequencer.add_track` in `docs/wiki-src/sequencer.md`. Tests
  (`Tests/Sequencer/TestTrackLookupEmptyName.cpp`):
  `PinWright.Sequencer.TrackLookup.EmptyNameMatchesNoTrack` (omitted and empty name on all five
  verbs -> `TRACK_NOT_FOUND`, both tracks survive unmuted with no sections) and
  `PinWright.Sequencer.RemoveTrack.NamedRemovalIsUndoable` (named removal removes only that track,
  the removal is the top undo transaction, and `GEditor->UndoTransaction` restores it).
