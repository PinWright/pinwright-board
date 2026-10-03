---
id: E-sequencer-frame-unit-cross-method-docs
title: "sequencer.list_sections reports keys[].frame and section range in tick-resolution ticks with no unit, while add_keyframe and the Python key API speak display-rate frames"
status: OPEN
severity: Low
category: ergonomic
tags: [cross-method-unit-split, units, sequencer, add_sub_sequence, set_sub_section_range, set_properties, list_sections, python-scripting-channels, docs, discoverability]
encounters: 2
costly: 1
lastSeen: 2026-08-27T23:55:00.0000000+05:00
rice: [1, 2, 1, 1]
priority: 17
---

# sequencer.list_sections reports key frames and section ranges in ticks without saying so, while the write verbs it is documented to verify take display-rate frames

`sequencer.list_sections {includeKeys:true}` emits each key's time as `keys[].frame` in
**tick-resolution ticks**: `BuildChannelKeysJson` writes the raw `FFrameNumber` from
`GetTimes()` / `GetKeys()` (`Source/PinWright/Private/Utils/MovieSceneJsonUtils.h:404`,
`:460`). Each section's `range.start` / `range.end` is raw ticks too (`MakeFrameRangeObject`,
`MovieSceneJsonUtils.h:54-63`, emitted at `:524`). Neither the `includeKeys` param
description (`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:4313-4316`) nor
`docs/wiki-src/sequencer.md:19` names a unit. Line 19 also says the keys are there "so an
`add_keyframe` write can be verified against the exact frames".

That verify recipe fails on its own terms. `sequence.add_keyframe`'s `frame` is a
**display-rate** frame. Its description just says "Frame number for the keyframe"
(`SequenceHandler.cpp:2439`), and the handler converts the value with `DisplayFrameToTick`
(`:297`, `:2569`). So `frame:30` written at 30 fps / 24000 tick reads back as
`keys[].frame:24000`. UE's Python scripting-channel API
(`MovieSceneScriptingDoubleChannel.get_keys()[i].get_time().frame_number`) reports display
frames. It is the only way to retune an existing key, because `add_keyframe` duplicates keys
and there is no `remove_keyframe`. A script that matches `list_sections` ticks against those
keys matches nothing. It also reports nothing wrong. Repro (History `#2`): 0 of 30 edits were
applied, and every follow-up check still passed.

Other units are already labelled: `set_properties` (`:644-646`, "display-rate frame number"),
`add_sub_sequence` / `set_sub_section_range` (`:3994-3995`, `:4086-4087`, "tick
resolution") and `set_playhead`. The gap is the `list_sections` reader and the unqualified
`add_keyframe` `frame`.

**Fix:** In `docs/wiki-src/sequencer.md:19`, state that `keys[].frame` and `range` are
tick-resolution ticks. Give the conversion `displayFrame = tick * displayRate / tickResolution`
(`get_properties` returns both rates). Replace the "verify an `add_keyframe` write" sentence
with that conversion, and note that UE's Python scripting-channel key API uses display frames.
Also name the unit in the `includeKeys` param description and in `sequence.add_keyframe`'s
`frame` description (display-rate). Optionally add a `displayFrame` field next to each
`keys[].frame`.

**Acceptance:** `list_sections` docs and `includeKeys` description say "tick-resolution";
`sequence.add_keyframe` `frame` description says "display-rate"; sequencer.md gives the tick
to display-frame conversion where the add_keyframe verify recipe is.

**Workaround:** Read `tickResolution` and `displayRate` with `sequencer.get_properties`, then
divide each `keys[].frame` by `tickResolution / displayRate` before comparing it with display
frames.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a clean/done master-cinematic assembly task (focus `sequencer.add_sub_sequence`; 17 RPCs, zero retries, zero errors, plan_divergence none, outcome clean). Sub-section write methods `add_sub_sequence` / `set_sub_section_range` take tick-resolution frames (docs say `(tick resolution)`) while sibling writer `set_properties` takes display-rate frames (docs say only `frame`); the split is never cross-referenced, so a caller who learns "ticks" from the sub-section family and carries it into `set_properties` silently mis-sizes the master playback range by the tick-resolution factor with no error. Not a defect — the Judge's replay confirmed `set_properties` converts correctly (playbackEnd:180 -> 144000 ticks) and the agent self-resolved via a planned `get_properties` read; filed as the pure-discoverability residual: a reciprocal cross-reference note on the sub-section H3s + the sequencer.md overlay giving the tick<->displayFrame conversion. Distinct from `E-sequencer-property-unit-drift` (the now-fixed set_properties write bug, whose fix scope never names the sub-section methods); same docs-pairing family as `E-add-sync-marker-frame-vs-seconds`. Evidence: agent friction line "unit-convention split ... which I resolved via get_properties (tickRes 24000, 30fps)"; CallAnalyzer "a less careful caller would silently pass a tick value like 144000 to playbackEnd ... with no error."

- `#2-list-sections-ticks-vs-python-display-frames` `OPEN` reporter — Same unit split, one method
  further out, and this half costs a **silent no-op** rather than a mis-sized range.
  `sequencer.list_sections {includeKeys:true}` reports each key's time as
  `keys[].frame` in **tick-resolution ticks** (`0, 16000, 52000 … 480000` at tickRes 24000 /
  60 fps), with no unit qualifier on the field. The documented follow-on for editing those keys is
  UE's Python scripting-channel API — `add_keyframe` duplicates rather than replaces and there is no
  `remove_keyframe`, so `MovieSceneScriptingDoubleChannel.get_keys()[i].set_value(...)` is the only
  way to retune an existing key — and **that API reports the same keys' times in display frames**
  (`0, 40, 130 … 1200`). A caller who reads times out of `list_sections` and matches them against
  `key.get_time().frame_number.value` matches **nothing**.
  Encountered 2026-08-27 re-keying `/Game/Atlantis/Cine/LS_Atlantis_Flythrough` in
  `EAContentExamples58`: an edit script keyed on the ticks `list_sections` had just returned applied
  **0 of 30** edits, and every verification it ran afterwards still passed — key counts unchanged,
  key times unchanged, tangents unchanged, `mark_package_dirty` -> `True`,
  `save_asset(only_if_is_dirty=False)` -> `True`. Nothing in the response distinguished "wrote
  nothing" from "wrote everything"; only an explicit per-key before/after log caught it. The
  failure is silent in exactly the way the sub-section case in `#1` is, and it lands on the
  **reader** the caller reaches for first.
  Remedy fits this ticket's existing scope: name the unit on `list_sections`'s `keys[].frame` in
  `docs/wiki-src/sequencer.list_sections.md` (**tick-resolution ticks**), and add to the
  `sequencer.md` units note that the Python scripting-channel surface a caller is sent to for
  key edits speaks display frames, with the same `displayFrame = tick * frameRate.num /
  tickResolution.num` conversion. Adjacent, already filed:
  `B-sequence-add-keyframe-duplicates-existing-frame` (why the Python route is necessary at all)
  and `B-sequencer-section-range-display-frames-as-ticks`.
- `#3-rephrased` `OPEN` developer — Narrowed the old text. The sub-section/`set_properties` cross-reference half is resolved at 7230b41d: `set_properties` now labels `playbackStart`/`playbackEnd`/`lengthInFrames` as display-rate frames (`SequenceHandler.cpp:644-646`), and the sub-section methods already say tick resolution (`:3994-3995`, `:4086-4087`). The rewrite targets the `#2` gap, still open: `list_sections` emits raw ticks for `keys[].frame` (`MovieSceneJsonUtils.h:404`, `:460`) and `range` (`:54-63`) with no unit in the param description (`SequenceHandler.cpp:4313-4316`) or in `sequencer.md:19`. Line 19 also offers those ticks as the way to verify display-frame `add_keyframe` writes, and `add_keyframe`'s `frame` description has no unit (`:2439`). Severity unchanged (Low, docs).
