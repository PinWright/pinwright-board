---
id: B-sequencer-add-track-returns-unresolvable-name
title: "sequencer.add_track echoes the requested trackName back unapplied, and six verbs resolve tracks by that string"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, silent-wrong-data, identifier, readback]
encounters: 1
---

# sequencer.add_track echoes the requested trackName back unapplied

`sequencer.add_track` answered with `TrackName.IsEmpty() ? TrackType : TrackName`
(`Handlers/Sequencer/SequenceHandler.cpp:2928` at `b92ba268`) — the caller's own request read
back out. Nothing was ever read off the created track, and the optional `trackName` parameter
was never written to it: the file contained no `SetDisplayName` call, and `TrackName` was
referenced at exactly two lines, the read at `:2861-2862` and that response line.

Six verbs resolve a track by that string — `add_section` (`:2249-2250`), `set_track_muted`
(`:2475`), `set_track_solo` (`:2555`), `set_track_locked` (`:2632`), `remove_track`
(`:2714-2715`), and the `list_sections` `trackName` filter (`:3321`) — so the documented next
step answered `TRACK_NOT_FOUND` for the exact string the previous call returned. An agent cannot
recover by re-reading the response, because the response *is* the wrong identifier.

**The failure condition is narrower than "always", and the reason it sometimes worked is an
accident.** With `trackName` omitted the response is `trackType`, and every resolver matches with
`FString::Contains`, so `"Audio"` happens to be a substring of the engine auto-name
`MovieSceneAudioTrack_0`. That breaks the moment `trackType` is spelled as anything that is not a
substring of the class name — e.g. the fully-qualified `/Script/MovieSceneTracks.MovieSceneAudioTrack`,
which `ResolveUClass` accepts. With `trackName` supplied it never resolved at all.

## The identifier had two incompatible definitions in one file

`UMovieSceneTrack::GetTrackName()` is `virtual FName GetTrackName() const { return NAME_None; }`
(`C:\UE_5.8\Engine\Source\Runtime\MovieScene\Public\MovieSceneTrack.h:390`) and is overridden by
exactly three engine classes (`UMovieScenePropertyTrack`, `UMovieSceneTimeWarpTrack`,
`UMovieSceneControlRigParameterTrack`). For every other track type it stringifies to the literal
`"None"`. `SequenceHandler.cpp` emitted `trackName` from that accessor in three places and from
`UObject::GetName()` in six, so:

- `sequencer.add_sub_sequence` (`:3027`) returned `trackName: "None"` — the same defect as
  `add_track`, an identifier that resolves nowhere.
- `sequencer.list_sections`' `trackName` filter compared against `"None"` with exact equality, so a
  `trackName` piped in from `list_tracks` or `add_track` matched zero sections and the verb
  reported an **empty `sections[]` for a track that has sections** — a lie by omission on the
  readback side of the same loop.

**Fix:** one definition of the identifier, used on both the emit and the resolve side, so the two
cannot drift.

## History
- `#1-found-during-lying-verb-sweep` `OPEN` reporter — Found by source audit at `b92ba268` while
  sweeping for verbs that report a value not matching what happened. Quoted above; the narrowing of
  the failure condition (substring matching makes the no-`trackName` branch resolve by accident)
  corrects the original triage note, which claimed the returned identifier never resolves.
- `#2-single-identifier-plus-apply-and-measure` `IN-REVIEW` developer — **Source only; NOT compiled
  and NOT runtime-verified** (a map agent held the editor DLL; an integration pass owns the build).
  All changes in `Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp` unless noted.
  - New file-scoped pair in `namespace SequenceHelpers`: `GetTrackIdentifier(const UMovieSceneTrack*)`
    returning `Track->GetName()`, and `TrackMatchesIdentifier(const UMovieSceneTrack*, const FString&)`
    matching `GetName()` OR `GetDisplayName()` with the pre-existing `Contains` semantics preserved
    verbatim (including empty-query-matches-first, deliberately unchanged). Every `trackName` emit
    and every `trackName` lookup in the file now routes through them; `GetTrackName()` no longer
    appears in the file outside comments.
  - `add_track` now APPLIES the requested name (`UMovieSceneNameableTrack::SetDisplayName`) instead
    of echoing it, and reports measured values: `trackName` = the track's object name, plus a new
    `displayName` field. Validate-before-mutate: a `trackName` on a class outside the
    `UMovieSceneNameableTrack` subtree is refused with the new `TRACK_NAME_NOT_APPLIED` **before**
    `AddTrack`, so nothing is left behind; a readback mismatch after the write removes the track and
    reports the same code. An unresolvable `trackType` is now `CLASS_NOT_FOUND` rather than
    `TRACK_CREATION_FAILED` ("Failed to add track of type: X" described an attempt that never happened).
  - `set_track_muted` / `set_track_solo` / `set_track_locked` now also match display name, which
    `remove_track` and `add_section` already did — needed because `add_track` now writes one.
  - `ERR_TRACK_NAME_NOT_APPLIED` registered in `Handlers/ErrorCodes.h` (the only new code).
  - Tests: `Source/PinWright/Private/Tests/Sequencer/TestAddTrackIdentifier.cpp` — 4 automation tests.
    Two assert the failure direction (`TRACK_NAME_NOT_APPLIED` on `UMovieSceneSpawnTrack` with the
    track count unchanged and a no-name control that still succeeds; `CLASS_NOT_FOUND` on a bogus
    type with nothing added). One pipes the returned identifier straight into `set_track_muted` and
    `set_track_locked` and asserts the graph really muted. One asserts the `list_sections` filter
    accepts the identifier the other verbs report. All four fail before the fix.
  - Docs: `Docs/wiki-src/sequencer.md` — new paragraphs under `### sequencer.add_track` separating
    request-`trackName` (a display name) from response-`trackName` (the resolvable identifier), plus
    a clause on the `list_sections` filter.
  - **For the tester:** the decisive live check is
    `sequencer.add_track {path, trackType:"Audio", trackName:"CameraMove"}` followed by
    `sequencer.set_track_muted {path, trackName:<the response's trackName>}` succeeding, where the
    same sequence pre-fix answered `TRACK_NOT_FOUND`.
- `#3-compiled-and-suite-green` `IN-REVIEW` developer — Supersedes `#2`'s "NOT compiled" caveat.
  Built and tested in integration pass 8; committed as `ad88ce0e` and pushed. Clean module rebuild
  (all 7 module intermediates moved aside, `-DisableAdaptiveUnity -NoHotReloadFromIDE`):
  `Result: Succeeded`, zero errors and zero warnings in both the build log and UBT's `-Log=` target,
  `SequenceHandler.cpp` present as a named compile action in its freshly generated unity blob
  (`Module.PinWright.17.cpp`), no standalone `.cpp` actions, all 7 DLLs relinked.
  Full suite **3723 tests performed, 3721 Success, 2 Fail** — the two pre-existing
  `localization.Validation.*` only; ZenServer probe 0; all five integration sub-modules loaded.
  All four `PinWright.Sequencer.AddTrackIdentifier.*` tests located by name in the log and
  `Result={Success}`.
  **Still not runtime-verified** — the four automation tests exercise the graph, but the live
  `add_track` → `set_track_muted` round trip in `#2`'s "For the tester" note has not been driven
  through the MCP surface. Stays `IN-REVIEW`; a tester still has to close it.
  Compiler note for whoever reads `#2` next: the whole `UMovieSceneNameableTrack` class body sits
  inside `#if WITH_EDITORONLY_DATA`, but PinWright's module is `"Type": "Editor"`, so the macro is 1
  in every configuration this plugin builds in. The clean compile is the proof — a 0 would have been
  a compile error at both the `SetDisplayName` call and the `IsChildOf` gate, not a silent
  degradation. No version guard was added or needed.
  Found while verifying, filed separately as `B-track-name-none-in-widget-anim-and-dump`: three live
  `GetTrackName()` emit sites survive OUTSIDE `SequenceHandler.cpp` and still emit `"None"`.
