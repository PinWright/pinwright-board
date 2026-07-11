---
id: E-sequence-add-keyframe-bare-empty-no-success
title: "sequence.add_keyframe (frame form) returns a bare {} with no echo of the written keyframe, forcing a list_sections readback to confirm it landed"
status: OPEN
severity: Low
category: ergonomic
tags: [sequencer, add_keyframe, no-echo, readback, round-trip]
encounters: 5
lastSeen: 2026-07-11T11:53:38.4703980+03:00
---

# `sequence.add_keyframe` (frame-numbered form) returns a bare `{}` — no echo of what was written

The frame-numbered `sequence.add_keyframe` writer (the legacy `SequenceHandler`
form used for transform/vector/generic-typed tracks) returns an **empty JSON
object `{}`** on success — no echo of the frame/property/binding/channel(s)
written. The write *does* succeed and *is* signalled (`isError:false` at the MCP
envelope — the audited calls were each `ok=true`); the gap is that the payload
carries no detail of *what* changed, so a caller cannot tell from the response
which channels/how many keys landed. The natural reaction — and the one the
audited task took — is to spend an extra `sequencer.list_sections` readback purely
to confirm the keyframe.

Every other authoring verb on the board's convention echoes what it wrote. In
particular the **modern seconds-based `sequencer.add_keyframe`**
(`SequencerHandler.cpp:141-147`) returns `AddAssetVerification(asset)` +
`bindingGuid`/`propertyName`/`time`/`value`; the legacy frame-numbered form should
align to that shape.

This is the response-shape ergonomic gap, distinct from two neighbor tickets on
the same method:

- `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW) is about
  `property="Location"` being *rejected*; this ticket is about the *successful*
  path returning a bare `{}`.
- `F-sequencer-transform-section-range-not-expanded-by-keyframe` (OPEN) is about
  the section range staying collapsed; this ticket is about the missing echo that
  *forces the readback in the first place*.

Note (corrected framing): the bare `{}` does **not** defeat success detection.
Success is already signalled by `isError:false`, and a genuine no-op is
distinguishable — when nothing matched, all branches fall through to
`Ctx.SendError("UNSUPPORTED_PROPERTY", ...)` (`SequenceHandler.cpp:1910`,
`isError:true`), so `{}`+`isError:false` is emitted *only* when a key was really
written. The friction is purely the missing detail-echo, not a broken/absent
`success` field (a literal `success:true` boolean would just duplicate
`isError:false`).

## Evidence (this task)

SEED-mode `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls).
Two `sequence.add_keyframe` calls (Transform property, frames 0 and 120) each
`ok=true` but each "returned bare `{}`", which the run noted forced a verification
readback. Verbatim friction note: "it returned a bare {} with no confirmation of
what was written, forcing a list_sections readback to verify keys landed." All
calls succeeded; the friction is the missing echo, not a failure.

## What to do

1. **Emit an echo of what was written (primary).** Have the frame-numbered
   `sequence.add_keyframe` return, on each success exit, an object echoing the
   write — asset verification (`AddAssetVerification`) plus the resolved
   `bindingId`, `property`, `frame`, and `tickFrame` — matching the modern
   seconds-based `sequencer.add_keyframe` response shape, so the caller confirms
   the write from the response instead of a `sequencer.list_sections` round-trip.
   Do **not** add a redundant `success:true` boolean; success is already
   `isError:false`.
2. **Wiki note (downstream wiki process, cheap).** On the `sequencer.add_keyframe`
   H3 in `docs/wiki-src/sequencer.md`, document the frame-numbered form's success
   response shape (the echo fields above) so a caller reads them instead of
   budgeting a `sequencer.list_sections` readback.

**Workaround (before the fix):** treat a bare `{}` from the frame-numbered
`sequence.add_keyframe` as success (`isError:false`), and if you must verify,
read back with `sequencer.list_sections` (check the target channel's `keyCount`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the SEED-mode
  `IntroFlythrough` cinematic task (`sequencer.create` focus, 17 calls; judge
  filed the section-range gap). This ticket covers a distinct PROCESS surface
  from the friction note: the frame-numbered `sequence.add_keyframe`
  (`property="Transform"`) returned a bare `{}` on both keyframe calls (frames 0
  and 120), forcing a `sequencer.list_sections` readback to confirm the keys
  landed. Dedup: ripgrep across OPEN/DONE/WONTFIX — distinct from
  `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW; that's the
  `property="Location"` *rejection*, this is the *successful* path's empty
  response) and from `F-sequencer-transform-section-range-not-expanded-by-keyframe`
  (OPEN; that's the section range, this is the missing echo that triggers the
  readback). The wiki `add_keyframe` H3 documents the dual call shape but never
  the bare-`{}` return. Sibling docs/process angle to
  `E-sequencer-set-track-state-no-readback-doc` (wasted verification probes from
  an unsignalled read-gap). Primary ask: emit an echo from the frame-numbered
  writer; interim: a wiki note on the `add_keyframe` H3.
- `#2-liveness` `OPEN` reporter — still observed. Struggle-audit of the
  `sequencer.remove_track` cinematic task (`IntroEstablishingShot`, 18 calls):
  both `sequence.add_keyframe` calls (Location, frames 0 and 120) returned a bare
  `{}` again, and the agent again spent an extra `sequencer.list_sections`
  (`includeKeys:true`) round-trip purely to confirm the two Location keys landed
  (SAY: "Let me verify the keyframes actually landed on the transform track" ->
  `keyCount:2`). Same symptom, same forced readback; no new angle.
- `#3-liveness` `OPEN` reporter — still reproduces at HEAD (SEED-mode `sequencer.delete` cinematic task, `IntroMaster`; replayed `sequence.add_keyframe` Location frame 45 → bare `{}`, same forced `sequencer.list_sections` readback).
- `#4-liveness` `OPEN` reporter — still observed (`CS_Establishing` 24fps 0-5s establishing-shot task, focus `sequencer`, 31 calls): both `sequence.add_keyframe` Location calls (frames 0 and 120) returned a bare `{}`, and the agent again ran a `sequencer.list_sections{includeKeys}` readback after EACH to confirm the key landed (2 extra RPCs). Same symptom, same forced readback; no new angle.
- `#5-reword-implement` `IN-REVIEW` developer — Reworded to match source: the title/body/primary-ask now target an ECHO (not a "success field"). Removed two verified-false framings — success is already `isError:false` (McpTransport.cpp:69-70), and a no-op returns `UNSUPPORTED_PROPERTY`/`isError:true` (SequenceHandler.cpp:1910), so success and no-op are already distinguishable; a `success:true` boolean would duplicate `isError:false`. Implemented the fix: all four success exits of the frame-numbered `sequence.add_keyframe` (Transform, per-axis Location/Rotation/Scale, generic float, generic bool) now return a `MakeKeyframeEcho` payload — `AddAssetVerification(LevelSeq)` + resolved `bindingId`/`property`/`frame`/`tickFrame` — mirroring the modern `sequencer.add_keyframe` (SequencerHandler.cpp:141-147), replacing `SendSuccess(nullptr)`. File: `Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`. Regression test: `Plugins/PinWright/Source/PinWright/Private/Tests/Sequencer/TestKeyframeEchoesWrittenPayload.cpp` (`PinWright.Sequencer.AddKeyframe.LocationEchoesWrittenPayload` / `.TransformEchoesWrittenPayload` / `.FloatPropertyEchoesWrittenPayload`) — drives the real handler via InvokeHandlerWithCapture on an in-code bound possessable and asserts the success Result echoes property/frame/tickFrame + assetPath; reverting to `SendSuccess(nullptr)` makes Capture.Result null and fails. Left the sibling playback verbs' `SendSuccess(nullptr)` (SequenceHandler.cpp:1276/1310/1353) untouched — those are `E-sequencer-playback-control-bare-empty-no-state`.
- `#6-attempt-failed` `OPEN` developer — Auto-fix attempt reached IMPL-UNVERIFIED; reverted and NOT pushed (build/tests not green).
- `#7-liveness` `OPEN` reporter — still reproduces at HEAD (`IntroCutscene` 24fps film-precision cutscene, focus `sequencer.set_tick_resolution`, 26 calls): both `sequence.add_keyframe` `Transform` calls (frames 0 and 120) returned a bare `{}`, so the run spent a `sequencer.list_sections{includeKeys}` verify readback (Loc.X 0->800, section `[0,300000]`, keyCount 2) purely to confirm the keys landed. Same symptom, same forced readback; no new angle. (The IN-REVIEW echo fix `#5` is not yet green per `#6`, so HEAD still emits `{}`.)
