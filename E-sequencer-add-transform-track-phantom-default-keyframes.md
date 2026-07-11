---
id: E-sequencer-add-transform-track-phantom-default-keyframes
title: "sequencer.add_transform_track hardcodes hasDefaultKeyframes:true in its success payload, but the section it creates is a zero-length [0,0] range with keyCount:0 on every channel — the field is a hardcoded lie"
status: OPEN
severity: Medium
category: ergonomic
tags: [hardcoded-field, sequencer, add_transform_track, readback, transform-track, silent-wrong-data]
encounters: 1
lastSeen: 2026-07-11T11:53:38.4703980+03:00
---

# `sequencer.add_transform_track` returns `hasDefaultKeyframes:true` but writes zero keys

`sequencer.add_transform_track` reports `hasDefaultKeyframes:true` in its success
payload, but the section it creates has **no keys at all**. The handler
(`SequencerHandler.cpp:716-728`) only does `AddTrack<UMovieScene3DTransformTrack>`
→ `CreateNewSection()` → `AddSection()`; it never seeds a keyframe. The response
field at `SequencerHandler.cpp:734` is set **unconditionally**:

```
Resp->SetBoolField(TEXT("hasDefaultKeyframes"), true);
```

There is no code path where it is `false`, and nothing in the handler writes a
default key, so the claim is a hardcoded value with no relationship to reality. A
caller who trusts `hasDefaultKeyframes:true` — the natural reading being "the track
already ships with default keys, I can skip seeding a rest pose" — is trusting a
lie. In the audited task the very next `sequencer.list_sections{includeKeys:true}`
readback showed the freshly created section as a **zero-length `[0,0]` range with
`keyCount:0` on all 10 channels** (9 `MovieSceneDoubleChannel` + 1
`MovieSceneFloatChannel`, `keys:[]` everywhere), directly contradicting the field.
The agent flagged the contradiction aloud ("the section is zero-length (0-0) with
no keys").

## What it should do

Make the field reflect reality, or drop it:

- Set `hasDefaultKeyframes:false` (or better, echo the actual `keyCount`, which is
  `0` here) when no keys were written; **or**
- if the intent is that a transform track should ship with a default rest-pose key,
  actually seed one at frame 0 and only then report `true`; **or**
- remove `hasDefaultKeyframes` entirely — a `success:true` + `bindingGuid` echo is
  already enough, and a hardcoded boolean that never varies carries no information
  and actively misleads.

At minimum the payload must not assert default keyframes exist when the section it
just created has none.

## Evidence (this task)

Struggle audit of a clean/`done` film-precision cinematic task (`IntroCutscene`,
focus `sequencer.set_tick_resolution`, 26 logged calls, zero execution errors). Call
sequence:

- `sequencer.add_transform_track {bindingGuid:D84FA9...}` → `{...,success:true,
  hasDefaultKeyframes:true}`.
- immediately after, `sequencer.list_sections {includeKeys:true}` → one
  `MovieScene3DTransformTrack` section, range `[0,0]`, `keyCount:0` on every one of
  the 10 channels, `keys:[]`.

The CallAnalyzer ground-truth trace flagged this: "add_transform_track reports
`hasDefaultKeyframes:true` in its success payload, but the section it creates is a
zero-length `[0,0]` range with `keyCount:0` on every one of the 10 channels … A
caller trusting `hasDefaultKeyframes:true` could conclude the track is already
keyed and skip authoring keys." The run went on to author real keys via
`sequence.add_keyframe`, so no wrong asset resulted here — but the field itself is
provably false on the normal path.

## Relationship to existing tickets

- Same method as `E-sequencer-add-transform-track-section-model-undocumented` but a
  **different root cause**: that ticket is the *docs* gap on the section/keying
  model (where keys attach, whether two keyframes split into two sections); this is
  a *hardcoded-wrong return field*. Per the dedup "same method AND same root cause →
  merge; else file separately" rule, filed separately and cross-linked.
- Sibling of the hardcoded/phantom readback-field family:
  `B-spawn-category-silent-noop-fake-existsafter` (hardcoded `existsAfter:true`),
  `E-session-info-readback-hardcoded` (hardcoded session fields),
  `E-variable-readback-instanceeditable-always-true` (hardcoded `instanceEditable:
  true`) — same `hardcoded-field` symptom, different method.

severity rationale: impact=hardcoded-wrong data on a normal path, caller trusts a
lie (High-class) × reach=cinematic transform-track authoring is a specific path,
not every-session (bump DOWN) → Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit (PROCESS) of the SEED-mode
  film-precision `IntroCutscene` task (focus `sequencer.set_tick_resolution`, 26
  logged calls, outcome tool_bug for a SEPARATE surface — the
  `set_tick_resolution` substring parse, judge-filed
  `B-sequencer-tick-resolution-substring-parse`). This ticket covers the distinct
  PROCESS surface flagged by the CallAnalyzer ground-truth trace:
  `sequencer.add_transform_track` returned `hasDefaultKeyframes:true` while the
  section it created read back as `[0,0]` with `keyCount:0` on all 10 channels.
  Confirmed against source — `SequencerHandler.cpp:734` sets the field
  unconditionally and the handler (`:716-728`) seeds no key. Dedup: ripgrep across
  OPEN/IN-REVIEW/DONE/WONTFIX — no ticket covers `add_transform_track`'s return
  field; distinct root cause from `E-sequencer-add-transform-track-section-model-undocumented`
  (docs/section-model, same method) and shares the `hardcoded-field` family with
  `B-spawn-category-silent-noop-fake-existsafter` / `E-session-info-readback-hardcoded`
  / `E-variable-readback-instanceeditable-always-true` (different methods). Ask:
  make the field reflect the real key count (0), seed an actual default key, or drop
  the field.
