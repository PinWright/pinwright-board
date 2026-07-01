---
id: F-sequencer-list-channels-keyframe-readback
title: "No live readback of per-key frame numbers/values on a sequencer channel — only an aggregate keyCount, so authored keyframes are unverifiable"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, add_keyframe, list_sections, readback, round-trip, list_channels]
---

# No live readback of per-key frame numbers/values on a sequencer channel — only an aggregate `keyCount`

After authoring keys with `sequence.add_keyframe`, a caller cannot verify *which
frames* were keyed or *what values* they carry through any sequencer readback.
The only channel-level read surface, `sequencer.list_sections`, emits per channel
just `{type, keyCount}` — never the per-key frame numbers or the per-key values.
So a task that keys a Location at frame 0 = (0,0,0) and frame 150 = (0,0,300), and
a Rotation at yaw 0 / yaw 360, can confirm only that *two keys exist on the
channel* (`keyCount=2`), not that they sit at frames 0/150 nor that they hold the
requested z/yaw values. The authored animation is structurally present but its
actual key data is opaque to the MCP.

This is the **deferred-consumer** case that the DONE `F-rpc-sequencer-list-sections`
explicitly parked: that ticket's body says *"Channels could alternatively split
into a separate `sequencer.list_channels` … Defer the split until a real consumer
needs per-channel detail beyond `{type, keyCount}` (e.g. enumerated key
times/values), at which point `list_channels` becomes the natural home for the
deeper payload."* This task is that real consumer — the per-key frame/value
detail is now needed to verify a keyframe-authoring round-trip.

Distinct from the three neighbor sequencer-keyframe tickets:

- `F-sequencer-transform-section-range-not-expanded-by-keyframe` (OPEN) — the
  *section range* stays collapsed; this ticket is about reading back the *keys
  themselves* (frame + value), an orthogonal read-side gap.
- `E-sequence-add-keyframe-bare-empty-no-success` (OPEN) — the writer returns a
  bare `{}`; this ticket is about the *reader* exposing no per-key data even when
  you do read back.
- `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW) — the
  `property="Location"` *write* path; this is the *read* path.

Sibling in another namespace: `E-rpc-animation-bone-track-readback` is the same
shape of gap (per-bone-track keys unreadable, only `rawTrackCount`) for the
animation surface — the sequencer surface has the identical hole at the channel
level.

## Root cause (read in source)

`sequencer.list_sections` → `SequenceHelpers::BuildListedSectionJson`
(`SequenceHandler.cpp`) wraps `MovieSceneJsonUtils::BuildSectionJson`
(`Utils/MovieSceneJsonUtils.h`), whose channel entries are
`{type, keyCount}` only — it counts keys (`Channel->GetNumKeys()` /
`ChannelData.GetTimes().Num()`) but never enumerates `GetTimes()` /
`GetValues()`. No other `sequencer.*` read RPC surfaces channel key data either.
The data exists on the channels (`FMovieSceneDoubleChannel::GetData().GetTimes()`
and `.GetValues()`), it is simply never serialized.

## Evidence (this task)

SEED-mode `LogoIntro` cinematic task (seed `sequence.add_keyframe`, 17 calls).
Four `sequence.add_keyframe` calls (per-axis form: Location frame 0/150, Rotation
frame 0/150) all `ok=true`; the post-write `sequencer.list_sections` reported the
transform section's Location/Rotation channels with `keyCount=2` each — but no
frame numbers and no values. Verbatim friction note:

> list_sections emits only channel type + keyCount, never per-key frame numbers
> or values, so the requested keys at frame 0/150 and the z:0->300 / yaw:0->360
> values are NOT verifiable through any readback.

The run's own SUCCESS-CHECK note: *"per-key frame/value data is not observable
through any sequencer readback."* All calls succeeded; the gap is purely that the
authored key data cannot be read back to confirm correctness.

## What it should do

Surface per-key data on the channel readback. Either:

1. **Extend `sequencer.list_sections` channel entries** — add an optional
   `keys: [{ frame: <frameNum>, value: <number|object> }, ...]` array (gated by
   an `includeKeys`/`keys=true` param to keep the default payload small), built
   from each channel's `GetData().GetTimes()` / `GetValues()`. Keeps the
   round-trip count down and the dump-parity story intact (the same enrichment
   would flow into `MovieSceneJsonUtils::BuildSectionJson` and thus
   `level_sequence.json`). Bump the dump aspect version if the dump shape grows.
2. **Add the deferred `sequencer.list_channels`** (params: section identity by
   `bindingGuid` + `trackName` + `rowIndex`; optional `channelIndex`/`type`
   filter) returning `{ channels: [{ type, keyCount, keys: [{frame, value}] }] }`
   — the home `F-rpc-sequencer-list-sections` named for the deep per-key payload.

Either makes the keyframe round-trip verifiable end-to-end (write → read the
exact frames + values back). Reuse the existing `MovieSceneJsonUtils` channel
helpers so the live RPC and the asset dump emit byte-identical key JSON.

**Workaround (today):** none through the sequencer read surface — the only path
is `python.execute` reading `Channel.get_data().get_times()` /
`.get_values()` directly, i.e. bypassing the MCP's sequencer readback entirely.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the SEED-mode
  `LogoIntro` cinematic task (seed `sequence.add_keyframe`, 17 calls; judge filed
  the section-range gap `F-sequencer-transform-section-range-not-expanded-by-keyframe`,
  whose `#3` covers this same task). This ticket is a distinct PROCESS surface:
  the friction note's verifiability gap — `sequencer.list_sections` exposes only
  `{type, keyCount}` per channel, never per-key frame numbers or values, so the
  task's keys (Location frame 0/150 z:0->300, Rotation frame 0/150 yaw:0->360)
  could not be verified through any sequencer readback (the run's SUCCESS-CHECK
  flagged exactly this: "per-key frame/value data is not observable through any
  sequencer readback"). Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX — no
  ticket requests sequencer per-key/`list_channels` readback. This is the
  real-consumer trigger the DONE `F-rpc-sequencer-list-sections` deferred
  ("enumerated key times/values" → `list_channels`). Distinct from
  `F-sequencer-transform-section-range-not-expanded-by-keyframe` (section range,
  not key data), `E-sequence-add-keyframe-bare-empty-no-success` (write response
  shape, not the reader), and `B-sequence-add-keyframe-location-property-rejected`
  (write path). Same-shape sibling in another namespace:
  `E-rpc-animation-bone-track-readback`. Primary ask: add per-key `{frame,value}`
  to `list_sections` channels (or the deferred `list_channels`) so the keyframe
  round-trip is verifiable.
- `#2-fix` `IN-REVIEW` developer — Implemented option 1: enriched the shared
  channel JSON helper with an opt-in `includeKeys` flag. `BuildChannelEntriesJson`
  (`Private/Utils/MovieSceneJsonUtils.h`) now takes `bIncludeKeys` (default
  `false`); when on, each channel also carries `keys: [{frame, value?}]`. Frame
  numbers come from the base-class virtual `FMovieSceneChannel::GetKeys()` (works
  for every channel type); the scalar `value` is surfaced for double/float/bool/
  integer channels via the new `TryGetChannelKeyValue` (typed `GetData().GetValues()[i]`),
  and omitted for channel types with no plain scalar. New `BuildChannelKeysJson`
  builds the per-key array. The flag threads through `BuildSectionJson` /
  `BuildTrackJson` (same file), `SequenceHelpers::BuildListedSectionJson`
  (`Private/Handlers/Sequencer/SequenceHandler.cpp`), and the
  `sequencer.list_sections` handler, which gains an `includeKeys` (default
  `false`) param read from the payload and forwarded down. Because the default is
  off, the asset-dump path (`LevelSequenceDumpBuilder.cpp` → `BuildTrackJson(Track)`)
  and all other existing callers emit byte-identical output — **no aspect-version
  bump needed**. Wiki overlay (`docs/wiki-src/sequencer.md`) documents the new
  param. Regression test
  `EditorAutomationRpcGateway.sequencer.list_sections.IncludeKeysReturnsPerKeyData`
  (`Private/Tests/Media/TestSequencerHandlers.cpp`) authors two float keys (frame
  0=0.0, frame 150=300.0) on a registered LevelSequence, calls the production
  `sequencer.list_sections` handler with `includeKeys=true`, and asserts the exact
  per-key `{frame, value}` come back — it would fail if the enrichment were
  reverted (the old payload had no `keys[]`). The existing
  `list_sections.ReturnsDumpParityFields` test was tightened to assert `keys` is
  absent by default. Distinct from the section-range, write-response-shape, and
  write-path neighbors. This closes the deferred-consumer case parked by the DONE
  `F-rpc-sequencer-list-sections`.
