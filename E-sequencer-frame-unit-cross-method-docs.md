---
id: E-sequencer-frame-unit-cross-method-docs
title: "sequencer sub-section methods take TICK-resolution frames while set_properties takes DISPLAY-rate frames — the split is un-cross-referenced, so a master-assembly caller who learns 'ticks' carries the wrong unit into set_properties"
status: OPEN
severity: Low
category: ergonomic
tags: [cross-method-unit-split, units, sequencer, add_sub_sequence, set_sub_section_range, set_properties, list_sections, python-scripting-channels, docs, discoverability]
encounters: 2
lastSeen: 2026-08-27T23:55:00.0000000+05:00
---

# sequencer frame-unit split across sibling write methods is not cross-referenced in the docs

Assembling a master level sequence out of sub-sequences forces the caller to
speak **two different frame units** to sibling `sequencer.*` write methods in one
session, and nothing in the docs pairs them:

- `sequencer.add_sub_sequence` and `sequencer.set_sub_section_range` take
  `startFrame` / `durationFrames` in **tick-resolution frames** — their param
  docs do say `(tick resolution)`.
- `sequencer.set_properties` takes `playbackStart` / `playbackEnd` /
  `lengthInFrames` in **display-rate frames** on input (and echoes/reads back in
  ticks). Its param docs today just say `Playback start frame` /
  `Playback end frame` / `Length in frames from start` — no unit qualifier, so
  "frame" reads as the same frame the sub-section methods just used.

A caller who lays out sub-sections in ticks (e.g. `startFrame=0/48000/96000`,
`durationFrames=48000` at tickResolution 24000 / 30fps) and then reaches for
`set_properties` to size the master's playback range naturally carries the tick
value forward (`playbackEnd:144000`), when the correct display-frame value is
`180`. There is **no error** on the wrong path — the range just silently comes
out ~1000x off. This works, but it is non-obvious: the two unit systems live on
adjacent methods in the same namespace with no note that they differ.

This is **not** a functional defect — the Judge's live replay confirmed
`set_properties` correctly converts display frames to ticks (`playbackEnd:180`
stored `144000` ticks), and in this task the agent navigated the split cleanly
by reading `get_properties` first (tickRes 24000, 30fps) and verifying the
readback. It is a pure discoverability hazard for a **less careful** caller who
skips that read.

## What it should do

Add a reciprocal cross-reference so the split is discoverable from either side,
on `docs/wiki-src/sequencer.md` and the affected method H3s:

- On the `add_sub_sequence` / `set_sub_section_range` overlay H3s: a note that
  these frames are **tick-resolution**, and that the playback-range writer
  `set_properties` (and `set_view_range`, which takes seconds) use **different**
  units — do not carry a tick value into them. Give the conversion
  `displayFrame = tick * frameRate.num / tickResolution.num` (and its inverse).
- On the `set_properties` H3: name the unit explicitly as
  **display-rate frame number** (this half overlaps the docs annotation already
  in the fix scope of `E-sequencer-property-unit-drift`; scope the sub-section
  cross-reference here to avoid duplicating that ticket's write-bug work).

## Evidence

Struggle audit of a clean/done master-cinematic assembly task (focus
`sequencer.add_sub_sequence`; create RC_Master + 3 shot sequences, place three
contiguous sub-sections on the master sub-track, retime the chase later with a
reduced timeScale, extend the master playback range). 17 real RPCs, **zero
retries, zero errors**, `plan_divergence: none` — an exceptionally clean trace
whose single friction was this unit split.

Agent friction line, verbatim: "the one wrinkle was a unit-convention split
(add_sub_sequence/set_sub_section_range use tick-resolution frames,
set_properties uses display-rate frames) which I resolved via get_properties
(tickRes 24000, 30fps) and verified the readback."

CallAnalyzer (ground-truth trace): "add_sub_sequence startFrame=0/48000/96000,
durationFrames=48000 ... But set_properties takes playbackEnd in DISPLAY-rate
frames: it passed playbackStart=0 playbackEnd=180 and the SAME call returned
playbackEnd:144000 in TICKS ... a less careful caller would silently pass a tick
value like 144000 to playbackEnd — clipping/expanding the master's playback range
by a factor of the tick resolution with no error." Rated it a low-severity
discoverability/docs hazard, "not a functional bug".

## Relationship to existing tickets

- Distinct from `E-sequencer-property-unit-drift` (IN-REVIEW, category bug): that
  is the set_properties **write bug** (storing raw ticks instead of converting),
  now fixed and replay-confirmed. This ticket is the **docs cross-reference** for
  the sub-section authoring methods (`add_sub_sequence` / `set_sub_section_range`),
  which that ticket's fix scope never names — filed separately per the "different
  method / root cause -> new ticket" dedup rule rather than re-inflating a narrowed
  bug ticket.
- Same docs-pairing family as `E-add-sync-marker-frame-vs-seconds` (a write verb
  whose unit differs from its paired reader and needs a docs pairing note) and
  `E-volume-set-extent-units-class-dependent-docs` — a units/discoverability docs
  gap, not a defect.

severity rationale: impact=docs/discoverability (Low) x reach=master-sequence
assembly is a specific cinematics path, not every-session -> Low.

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
