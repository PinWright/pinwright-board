---
id: B-sequence-add-keyframe-duplicates-existing-frame
title: "sequence.add_keyframe APPENDS a second key at a frame that already has one instead of replacing it — re-keying a pose to fix framing silently leaves two conflicting keys at the same tick, and the verb returns {} so nothing says so"
status: OPEN
severity: High
category: bug
tags: [sequencer, add_keyframe, transform-track, duplicate-keys, silent-corruption, no-echo, iteration, cinematics, curve-integrity]
encounters: 2
lastSeen: 2026-08-27T22:45:00+05:00
---

# Re-keying an existing frame corrupts the curve instead of updating it

Calling `sequence.add_keyframe` with a `frame` that already carries a key **adds a second key at the
same tick**. The old key is not replaced and not removed. The channel ends up with two keys at one
time holding different values, which is not a state Sequencer's UI can produce and not one any
caller asks for.

The verb returns a bare `{}` (`E-sequence-add-keyframe-bare-empty-no-success`), so there is no
signal at all — not a warning, not a count, not the resulting key.

## Repro, measured

On `/Game/Atlantis/Cine/LS_Atlantis_Flythrough` (60 fps, tick resolution 24000, so 400 ticks per
display frame), a `MovieScene3DTransformTrack` carrying 23 keys.

Four frames — 820, 860, 900, 940 — were re-keyed to new values to fix a camera framing problem.
Four calls, all `property:"Transform"`, all returning `{}`:

```
sequence.add_keyframe { frame: 820, value:{ location:{x:-1381,y:-5839,z:2400}, rotation:{pitch:6,   yaw:76.6,  roll:0}}}
sequence.add_keyframe { frame: 860, value:{ location:{x:2778, y:-5986,z:4400}, rotation:{pitch:-14, yaw:114.9, roll:0}}}
sequence.add_keyframe { frame: 900, value:{ location:{x:6237, y:-3167,z:4300}, rotation:{pitch:-13, yaw:153.1, roll:0}}}
sequence.add_keyframe { frame: 940, value:{ location:{x:6765, y: 1364,z:3600}, rotation:{pitch:-11, yaw:191.4, roll:0}}}
```

Saved, then **reloaded from disk** (`asset.reload`) so the readback cannot be the in-memory object,
then `sequencer.list_sections {includeKeys:true}`:

`keyCount` is **27, not 23**, on every one of the six animated channels. The X channel:

```
{"frame":328000,"value":-1151}   <- original key, frame 820
{"frame":328000,"value":-1381}   <- new key, SAME tick
{"frame":344000,"value":1852}    <- original key, frame 860
{"frame":344000,"value":2778}    <- new key, SAME tick
{"frame":360000,"value":5886}    ... and again at 900
{"frame":360000,"value":6237}
{"frame":376000,"value":6666}    ... and again at 940
{"frame":376000,"value":6765}
```

Four re-keys, four duplicate pairs, +4 keys per channel. The yaw channel is the tell that this is
duplication rather than anything subtler: frames 820/860/900/940 were re-keyed with the *same* yaw
values, and it still produced pairs — `{"frame":328000,"value":76.6}` twice, `{"frame":344000,
"value":114.9}` twice, and so on. Value equality does not deduplicate.

## Why it is High and not cosmetic

**It corrupts the evaluated curve, silently.** Two keys at one tick make the tangent solve and the
evaluated value at that frame undefined-by-construction; which one wins is an implementation detail
of the channel's key array order. In the measured case the *stale* value was first in the array.

**It breaks the single most common cinematic iteration.** Authoring a camera move is: key a rough
path, look at frames, fix the ones that read badly, look again. That loop is "re-key this frame".
Through this API that loop poisons the curve a little more on every pass, and the only symptom is
that the shot stops matching the numbers you just wrote — which reads as a *different* bug (a
tangent problem, an evaluation problem, a save problem) and sends the author hunting in the wrong
place. It cost a full track rebuild here, and the only reason it was caught at all is that the
`.uasset` byte count jumped +2159 after four writes that should have been value-only and
size-neutral.

**Byte size will not catch it in general.** A value-only re-key on a fixed-width double channel does
not move the file size, so the "verify the bytes grew" heuristic this project relies on reports
nothing either way. `sequencer.list_sections {includeKeys:true}` and a `keyCount` comparison is the
only reliable detector, and no caller thinks to run it after a *re-key*.

## Expected

`AddKey`-style semantics on a channel keyed by time: if a key already exists at that frame number,
**update it in place** (value, interp, tangent mode) and keep the key count constant. That is what
the Sequencer UI does when you set a key on a frame that has one, and it is what
`FMovieSceneDoubleChannel::AddCubicKey`'s `UpdateOrAddKey` counterpart is for
(`FMovieSceneChannelData::UpdateOrAddKey`, which searches by `FFrameNumber` before inserting).

The current code almost certainly calls the plain add/insert path. Both call shapes of the verb
should be checked, and the per-axis `Location`/`Rotation`/`Scale` forms as well as `Transform`.

## Suggested fix

1. Route every key write through `UpdateOrAddKey` (or an explicit "find by frame, overwrite else
   insert") on each affected channel. Constant key count on re-key is the invariant.
2. Regression test: key frame N, key frame N again with a different value, assert `keyCount`
   unchanged **and** the stored value is the second one. Reverting the fix must fail it. Do it on a
   transform track, since that is six channels and the per-channel loop is where a partial fix would
   hide.
3. Echo the result. Once `E-sequence-add-keyframe-bare-empty-no-success`'s echo lands, include
   whether the write **replaced** or **inserted** — that one word would have made this defect
   self-reporting rather than something found by accident four frames later.
4. Worth a `sequencer.remove_keyframe` (or `remove_keys {frameRange}`) alongside: there is currently
   **no verb that can delete a key**, so a caller who hits this has no repair route short of
   `remove_track` + `add_transform_track` + re-authoring every key. That is what this session had to
   do — 23 keyframe calls to undo 4.

## Impact

High. Silent curve corruption from the ordinary iteration loop, on the namespace's primary
authoring verb, with no response, no warning, no size signal, and no way to undo it short of
rebuilding the track.

## Distinct from

- `E-sequence-add-keyframe-bare-empty-no-success` (Low, 8 encounters) is the missing echo. It is the
  reason this defect is invisible, but the defect itself is the duplicate write, not the silence.
- `F-sequencer-transform-section-range-not-expanded-by-keyframe` is about the section *range*; the
  range was correct throughout here (`[0, 480000]`).
- `E-sequencer-add-transform-track-phantom-default-keyframes` is about keys appearing at track
  creation; these appeared at re-key time on an already-populated track.
- `B-sequencer-add-keyframe-seconds-as-ticks` (IN-REVIEW) is a time-unit fault. Units were correct
  throughout here: frame 820 landed at tick 328000 = 820 x 400 exactly.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Maps/Atlantis`, 2026-08-27. Sequence
`/Game/Atlantis/Cine/LS_Atlantis_Flythrough`, display rate 60, tick resolution 24000, playback
`[0, 1200]`. Binding `C35BAEA544B50124B4EA1B8B9313991C` (`ACineCameraActor` possessable), track
`MovieScene3DTransformTrack`.

## History

- `#1-measured` `OPEN` reporter — Hit while iterating a 20 s camera flythrough: four orbit keys
  re-keyed to fix a framing problem, `keyCount` went 23 -> 27 on all six channels. Confirmed after
  `asset.save {force:true}` **and** `asset.reload`, so the duplicates are on disk and not an
  in-memory artefact. Repaired by `sequencer.remove_track` on the transform track, re-adding it, and
  re-authoring all 23 keys; the rebuilt track reads back at exactly 23 keys per channel with the
  intended values.
- `#2-source-located` `OPEN` reporter — Confirmed the call site while retuning the same camera
  path. The transform branch writes with `Channels[n]->GetData().AddKey(TickFrame,
  MakeDoubleKey(v))` at `Handlers/Sequencer/SequenceHandler.cpp:2276-2329` and `:2416` — the plain
  insert, exactly as `#1` predicted. **The fix is already present in the same function**: the
  generic float and bool branches a few lines below use
  `Channel->GetData().UpdateOrAddKey(...)` at `:2469` and `:2508`. So this is three lines adopting
  the sibling branch's call, not new logic. Note the same three lines also carry
  `B-sequence-add-keyframe-transform-keys-never-auto-set-tangents` (default-constructed
  `FMovieSceneTangentData`, `AutoSetTangents()` never called) — one edit closes both, and both
  should get their regression tests in that edit.
  **A cheaper repair route than `#1`'s full track rebuild exists.** UE's Python sequencer scripting
  API has what the RPC surface lacks: `MovieSceneScriptingDoubleChannel` exposes `get_keys()`,
  `add_key`, and **`remove_key`**, and each returned key carries `set_value` / `set_time` /
  `set_tangent_mode` / `set_arrive_tangent` / `set_leave_tangent`. Driven through
  `python.execute` that is a read-modify-write over the existing curve. Verified on this sequence:
  ten keys re-valued across four channels (40 `set_value` calls) left `keyCount` at exactly 23 per
  channel, key times unchanged, and tangent mode/values untouched — so it neither duplicates nor
  perturbs anything else. Recommend citing it in the ticket as the interim workaround, since
  `remove_track` + `add_transform_track` + 23 re-keys is a large blast radius for what is usually a
  one-key fix.
