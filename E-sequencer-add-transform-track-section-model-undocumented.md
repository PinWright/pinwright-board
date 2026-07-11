---
id: E-sequencer-add-transform-track-section-model-undocumented
title: "sequencer.add_transform_track doc omits its section/keying model (all transform keys land on ONE section at tick 0, no split, no add_section) — forcing engine-source dives to confirm two keyframes won't split"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, sequencer, add_transform_track, transform-track, section-model]
encounters: 1
lastSeen: 2026-07-11T10:03:10.0692605+03:00
---

# `sequencer.add_transform_track` doc omits how keyframes attach to its section

The `sequencer.add_transform_track` overlay/page in `docs/wiki-src/sequencer.md`
documents only its params (`{sequencePath, bindingGuid}`) and that it adds a
transform track to a binding. It says **nothing about the keying model** that
follows — specifically:

- the track gets **one** `UMovieScene3DTransformSection`, and
- every subsequent transform keyframe (via the legacy frame-numbered
  `sequence.add_keyframe`) routes through the plugin helper
  `GetOrAddTransformChannels` → `FindOrAddSection(0)`, so **all** keys land on
  that **single** section regardless of frame — no per-key `add_section`, no
  split into two sections.

Absent that note, a careful caller cannot tell from the docs whether a
frame-0 + frame-120 keyframe pair will share one section or split across two (a
split would fail a "one section spans the shot" success check). In this task the
run reasoned aloud that "a naive frame-0 + frame-120 pair would split across two
sections and fail the check" and, to resolve the uncertainty, dropped into
**last-resort engine + plugin source reads** — `MovieScenePropertyTrack.cpp`
(`FindOrAddSection`/`FindSection`), `MovieSceneSection.cpp` (default range
`TRange(0)` = `[0,0]`), `MovieScene3DTransformSection.cpp`, and the plugin
`GetOrAddTransformChannels` (`SequenceHandler.cpp`) — purely to confirm both keys
share one section. ~6 source reads that a one-line docs note would eliminate.

This is a pure **discoverability** cost — the behavior works (the readback in
this same task showed a single transform section `[0,120000]` with 2 keys per
channel, exactly as desired). Nothing is broken; the model is just undocumented,
so a diligent caller pays it in engine-source reading.

## What it should do

**Wiki edit (downstream wiki process — name the page).** On the
`sequencer.add_transform_track` H3 in `docs/wiki-src/sequencer.md`, add a short
"keying model" note:

- The track owns a single transform section; all transform keyframes attach to
  that one section (no `add_section` needed, no split by frame).
- With the auto-expand behavior now in the keyframe writer (see
  `F-sequencer-transform-section-range-not-expanded-by-keyframe`, IN-REVIEW), the
  section grows to span every written key, so after keying at frames 0 and 120
  the section reads back `[0,120000]` ticks.
- Cross-link the correct keyframe method for transforms — the legacy
  frame-numbered `sequence.add_keyframe` with `property="Transform"` (or the
  per-axis `Location`/`Rotation`/`Scale`), not the float-only
  `sequencer.add_keyframe` (see
  `E-sequence-add-keyframe-per-axis-value-shape-undocumented`).

## Evidence

Struggle audit of a clean/`done` cinematic-blockout task (focus
`sequencer.create`; author `/Game/Cinematics/IntroFlyby`, add a camera + bound
transform track, key a 5s sweep, cut to camera, lock 24fps/5s). 12 RPCs, **zero
execution errors, zero retries** — the entire friction was pre-write discovery.
CallAnalyzer (ground-truth trace) flagged this as a `wiki-nav` inefficiency on
`sequencer.add_transform_track`: the page "does not explain the keying model …
Absent that note, a careful agent burned many engine-source reads
(MovieScenePropertyTrack, MovieSceneSection default [0,0] range, FindSection
semantics) fearing a two-keyframe split into two sections", resolved only after
reading `GetOrAddTransformChannels` and concluding "both keys land on the SAME
section, no split, no add_section needed."

Agent friction line, verbatim: "whether two keyframe calls land on one section
vs split), so I fell back to reading plugin C++ … and engine MovieScene source to
confirm GetOrAddTransformChannels uses FindOrAddSection(0)+ExpandToFrame so both
keys share one section."

## Relationship to existing tickets

- Distinct from `E-sequence-add-keyframe-per-axis-value-shape-undocumented`
  (different method `sequence.add_keyframe`; that ticket is the keyframe
  **value-object shape** + method-name-collision doc gap, not the transform
  track's section-attachment model).
- Distinct from `F-sequencer-transform-section-range-not-expanded-by-keyframe`
  (IN-REVIEW; that is the section-range **auto-expand defect/feature**, a
  behavior change — this is the **docs** gap on where keys attach, which the
  fixed behavior still needs explained). Cross-referenced above, not merged, per
  the "different method / root cause → new ticket" dedup rule.

severity rationale: impact=docs/discoverability (Low) × reach=cinematic
transform-keying is a specific path, not every-session → Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit (PROCESS) of the SEED-mode
  `sequencer.create` cinematic task (`/Game/Cinematics/IntroFlyby`, 24 logged
  calls / 12 RPCs, zero errors, outcome tool_bug for a SEPARATE surface —
  add_camera_track, judge-filed). This ticket covers the distinct PROCESS surface:
  `sequencer.add_transform_track`'s wiki page omits the section/keying model, so
  the run could not confirm from docs that a frame-0 + frame-120 transform
  keyframe pair lands on ONE section (vs splitting into two and failing the
  check) — it dropped into ~6 last-resort engine/plugin source reads
  (`MovieScenePropertyTrack` FindOrAddSection/FindSection, `MovieSceneSection`
  default `[0,0]` range, `MovieScene3DTransformSection`, plugin
  `GetOrAddTransformChannels`) to learn `FindOrAddSection(0)` routes all keys to
  the single section. Behavior works (readback: one transform section `[0,120000]`,
  2 keys/channel); the gap is pure discoverability. Dedup: ripgrep across
  OPEN/IN-REVIEW/DONE/WONTFIX — no ticket documents the add_transform_track
  section model; distinct from `E-sequence-add-keyframe-per-axis-value-shape-undocumented`
  (value-shape/method-collision on a different method) and
  `F-sequencer-transform-section-range-not-expanded-by-keyframe` (the range
  auto-expand defect, not a docs gap). Downstream wiki target:
  `docs/wiki-src/sequencer.md` `sequencer.add_transform_track` H3 — add a keying-
  model note + cross-links to the correct keyframe method.
