---
id: F-sequencer-batch-keyframes
title: "No batch keyframe write: a 20 s camera path is 23 separate sequence.add_keyframe RPCs, each returning {} — re-filed per F-sequencer-curve-channel-ops #2, which dropped the batch rider and asked for it separately when hit"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, keyframes, add_keyframe, batch, cinematics, camera-path, rpc-count, crash-exposure, atomicity]
encounters: 1
lastSeen: 2026-08-27T19:40:00+05:00
---

# One RPC per keyframe does not scale to a real camera move

`F-sequencer-curve-channel-ops` `#2` reworded itself down to the interp/tangent core and explicitly
dropped the batch `edit_keys` rider — *"re-file separately if hit"*. Hit, so here it is, with the
measurement the original ticket did not have.

Authoring one 20-second looping camera flythrough
(`/Game/Atlantis/Cine/LS_Atlantis_Flythrough`, 60 fps, frames 0-1200) took **23 separate
`sequence.add_keyframe` calls** on a single `MovieScene3DTransformTrack` binding. Each call writes
one frame across six channels (X, Y, Z, roll, pitch, yaw), and each returns a bare `{}`
(`E-sequence-add-keyframe-bare-empty-no-success`). 23 is not a heavy path — it is roughly one key
per beat plus four shaping keys. A hand-animated flythrough of the same length would routinely carry
60-100.

## Why this is more than an RPC-count complaint

**1. Each round trip is crash exposure, and the writes are not atomic.** This editor crashed five
times in the session the path was authored in, twice inside my own working window. The keys live in
memory until an explicit `asset.save {force:true}`, so a crash part-way through a 23-call sequence
leaves the sequence holding an arbitrary prefix of the path — a valid, loadable, silently
half-authored `ULevelSequence`. There is no way to make the path land or not land as a unit. A batch
verb makes the whole path one transaction, one failure point, and one save.

**2. The missing echo compounds with the call count.** Because each write returns `{}`, the only
proof is a `sequencer.list_sections {includeKeys:true}` readback. At one or two keys that is a cheap
verify-after-write. At 23 it is only affordable as a single check at the end — which means that if
anything goes wrong mid-batch you learn it late and cannot tell which call failed. One batch call
with one echo of what it wrote removes the whole dilemma.

**3. It forces the caller to reinvent the transform-value shape 23 times.** Every call repeats the
full `{location:{x,y,z}, rotation:{pitch,yaw,roll}}` envelope plus `path`, `bindingId`, `property`
and `interp` — about 200 bytes of boilerplate per key, all identical except the frame and six
numbers. A batch shape lets the invariant parts be stated once.

## Proposed shape

```js
sequencer.add_keyframes({
  path: "/Game/Atlantis/Cine/LS_Atlantis_Flythrough",
  bindingId: "9D3651C4497E5DA2BE143086173708E7",
  property: "Transform",
  interp: "cubic",                     // batch default, per-key override allowed
  keys: [
    { frame: 0,    value: { location:{x:-17500,y:0,z:9600},  rotation:{pitch:-22,yaw:0,roll:0} } },
    { frame: 40,   value: { location:{x:-17024,y:-152,z:8838}, rotation:{pitch:-21.05,yaw:0.48,roll:0} } },
    // ...
    { frame: 1200, value: { location:{x:-17500,y:0,z:9600},  rotation:{pitch:-22,yaw:360,roll:0} } }
  ]
})
```

Response should echo per key what the existing single-key echo work (`#5` on
`E-sequence-add-keyframe-bare-empty-no-success`) settled on — resolved `frame` **and** `tickFrame`,
`property`, `bindingId` — plus a `written` count and the resulting section range, so the caller can
confirm the batch landed without a second readback.

Requirements that matter:

- **One `FScopedTransaction` for the whole batch**, so undo is one step and a mid-batch failure
  rolls back rather than leaving a prefix.
- **All-or-nothing on validation.** Validate every key's frame and value shape before writing any of
  them; a bad key at index 17 must not leave 17 keys behind.
- Same two call shapes the singular verb has to live with (frame-numbered transform/vector vs
  seconds-based float) — batch the frame-numbered one first, since that is the one cinematics
  actually use (see `E-sequence-add-keyframe-per-axis-value-shape-undocumented`, six encounters, all
  of them falling back to the legacy frame form for exactly this reason).

## Impact

Medium. Nothing is unachievable without it — the 23 calls did work and the path is correct — but it
is the difference between authoring a cinematic and typing one out. It is also the cheapest
available mitigation for two problems the board already tracks separately: the missing write echo,
and the general fragility of long RPC sequences against an editor that does not stay up.

## History

- `#1-measured` `OPEN` reporter — Re-filed as invited by `F-sequencer-curve-channel-ops` `#2`.
  Measured on a real deliverable rather than a probe: 23 `sequence.add_keyframe` calls for one
  camera path, 6 channels each, confirmed landed by a single
  `sequencer.list_sections {includeKeys:true}` readback (23 keys per channel, section range
  `[0, 480000]` ticks). `interp:"cubic"` was accepted and echoed per key in that readback, so the
  interp half of `F-sequencer-curve-channel-ops` is live at HEAD — this ticket is only about the
  dropped batch rider. Two editor crashes occurred in the same working window, which is where the
  atomicity argument comes from rather than from theory.
- `#2-batch-verb-shipped` `IN-REVIEW` developer — "Added `sequencer.add_keyframes` in `Handlers/Sequencer/SequenceHandler.cpp`, the frame-numbered transform shape first as asked: `path` / `bindingId` / `actorName` / `property` / `interp` / `tangentMode` / `arriveTangent` / `leaveTangent` stated once, plus a required `keys[]` whose entries override those field by field. Every key is resolved and validated BEFORE the track or section is touched — `GetOrAddTransformChannels` is itself a mutation — so a bad entry at any index rejects the whole batch with INVALID_ARGUMENT naming `keys[i]` and leaves not even the track behind. The writes run inside ONE `FScopedTransaction`, with `Modify()` called BEFORE them so the single undo step actually restores. The response carries `written`, `channelsTouched`, `sectionRange` and a per-key echo of `frame`, `tickFrame`, `property`, `bindingId`, `interp` and `tangentMode`, so the confirming `list_sections` readback is no longer needed. It reuses `sequence.add_keyframe`'s per-key write path rather than forking it: both verbs now call `SequenceKeyframeHelpers::ParseTransformKeyValue` / `ApplyTransformKeyWrite` / `AutoSetTangentsOn`, which carry the update-or-add and solve-the-tangents contracts, and auto tangents are solved once per channel after the whole batch rather than per key on a partial curve. The seconds-based float form stays `sequencer.add_keyframe`. Covered by `PinWright.Sequencer.SequencerAddKeyframes.BatchWritesEveryKeyWithPerKeyOverrides` and `PinWright.Sequencer.SequencerAddKeyframes.InvalidKeyRejectsTheWholeBatch` in `Source/PinWright/Private/Tests/Sequencer/TestSequenceAddKeyframeInterp.cpp`. NOT compiled and NOT run here; a build and suite pass is the verification."
