---
id: B-track-name-none-in-widget-anim-and-dump
title: "GetTrackName() still emits the literal \"None\" as a trackName in widget.animation.* and the asset-dump path"
status: OPEN
severity: Medium
category: bug
tags: [sequencer, widget-animation, asset-dump, silent-wrong-data, identifier, readback]
encounters: 1
---

# Three `GetTrackName()` emit sites survive outside `SequenceHandler.cpp`

`B-sequencer-add-track-returns-unresolvable-name` established that
`UMovieSceneTrack::GetTrackName()` is the wrong accessor for a caller-visible track identifier:
it is `virtual FName GetTrackName() const { return NAME_None; }`
(`C:\UE_5.8\Engine\Source\Runtime\MovieScene\Public\MovieSceneTrack.h:390`) and is overridden by
exactly three engine classes (`UMovieScenePropertyTrack`, `UMovieSceneTimeWarpTrack`,
`UMovieSceneControlRigParameterTrack`). For every other track type it stringifies to the literal
`"None"`, which resolves in no `trackName` lookup anywhere in the plugin.

That fix (`ad88ce0e`) routed every emit and every lookup in `SequenceHandler.cpp` through
`SequenceHelpers::GetTrackIdentifier` / `TrackMatchesIdentifier`. It did **not** reach three live
call sites in other files, found while verifying it:

- `Source/PinWright/Private/Utils/MovieSceneJsonUtils.h:261` —
  `Obj->SetStringField(TEXT("name"), Track->GetTrackName().ToString());`
  This is the shared track-JSON builder, so the track-level `name` field is `"None"` for most track
  types wherever this helper is consumed — including the asset-dump `level_sequence.json` shape that
  `CLAUDE.md` says the live readers deliberately mirror. Note `SequenceHandler.cpp:206` calls
  `MovieSceneJsonUtils::BuildSectionJson` and then **overwrites** `trackName` at `:212`, so the
  *section* path happens to be covered; the track-level `name` is not.
- `Source/PinWright/Private/Handlers/UI/WidgetAnimationHandler.cpp:850`
- `Source/PinWright/Private/Handlers/UI/WidgetAnimationEventIntrospection.cpp:191`

`UWidgetAnimation` tracks are `UMovieSceneTrack` subclasses of the same kinds, so
`widget.animation.*` has the same defect one namespace over: a caller piping a reported track name
into the next call has a name that is `"None"`.

## Why it is filed separately rather than folded into the sequencer ticket

The sequencer fix is scoped to one file and is committed, tested and pushed. These three sites need
their own decision, because unlike the sequencer verbs the fix is not purely mechanical:

- `MovieSceneJsonUtils.h` feeds the **asset dump**, whose serialized bytes are cache-versioned. Per
  `CLAUDE.md` "Aspect Version Bumping", changing this field must bump the matching aspect entry in
  `Handlers/Asset/AssetDumpCache.cpp` in the same commit, or existing `.dumpcache.json` markers keep
  serving the old `"None"` output until callers pass `force=true`.
- The two `widget.animation.*` sites need their **resolve** side audited the same way the sequencer
  ones were — emitting a correct identifier is only half the loop if the lookups still match on
  something else.

## Repro (not yet run live)

Any level sequence or widget animation with a track type outside the three overriding classes —
e.g. an audio track or a sub-sequence track. Read it back through the asset dump or through
`widget.animation.*` and the track's `name` / `trackName` is the string `None`.

## History
- `#1-found-verifying-sequencer-identifier-fix` `OPEN` reporter — Found in integration pass 8 while
  verifying `B-sequencer-add-track-returns-unresolvable-name`'s fix at `ad88ce0e`; the verification
  grepped `Source/` for surviving `GetTrackName()` call sites and found these three outside the
  file that fix touched. Source audit only — **not** reproduced live, and no fix attempted.
