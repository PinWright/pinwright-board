---
id: E-sequence-add-keyframe-per-axis-value-shape-undocumented
title: "sequence.add_keyframe wiki documents the two call-shape param sets but not the per-axis value-object shape (flat {x,y,z}/{roll,pitch,yaw} vs nested {location,rotation,scale}), forcing source-reading"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, sequencer, add_keyframe, transform-track, value-shape]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `sequence.add_keyframe` value-object shape per property is undocumented — caller had to read plugin source

The `sequencer.add_keyframe` H3 in `docs/wiki-src/sequencer.md` does a good job
disambiguating the **method-name collision** — it explains the two registered
call shapes by *parameter set* (the seconds-based `SequencerHandler` float writer
vs the frame-numbered `SequenceHandler` broader-track writer). But for the
frame-numbered form it stops at the param list and never documents the **shape of
the `value` object per `property`**, which differs by branch:

- `property="Transform"` → **nested**: `value:{ location:{x,y,z}, rotation:{roll,
  pitch,yaw}, scale:{x,y,z} }` (any subset).
- `property="Location"` / `"Rotation"` / `"Scale"` (the per-axis branch) →
  **flat**: `value:{x,y,z}` (Rotation also accepts `{roll,pitch,yaw}`), or a bare
  `[x,y,z]` array.

These two value shapes are not interchangeable, and which one a given `property`
expects is not stated anywhere in the wiki. The audited task had to **read the
plugin C++** (`SequenceHandler.cpp` Transform branch vs the per-axis
Location/Rotation/Scale branch) to learn that `Location`/`Rotation` take the flat
`{x,y,z}`/`{roll,pitch,yaw}` shape while `Transform` takes the nested
`{location:{},rotation:{}}` shape. That is a discovery cost the wiki should
absorb.

This is the **docs counterpart** of the IN-REVIEW fix
`B-sequence-add-keyframe-location-property-rejected`: that ticket adds the
per-axis `Location`/`Rotation`/`Scale` branch (making the per-axis value path
work), but once it works, callers need to *know its value shape* — and the H3
documents neither the new per-axis shape nor the Transform-nested shape it
contrasts with. Distinct from `E-sequence-add-keyframe-bare-empty-no-success`
(that's the empty `{}` *response*; this is the *input* value-shape doc gap).

## Evidence (this task)

SEED-mode `LogoIntro` cinematic task (seed `sequence.add_keyframe`, 17 calls).
Four `sequence.add_keyframe` calls succeeded (per-axis Location/Rotation at frames
0/150), but only after the run learned the value shape from source. Verbatim
friction note:

> the add_keyframe wiki documents a confusing two-shape method-name collision
> (seconds-based sequencer form vs frame-numbered sequence form) — I had to read
> the plugin source to learn the flat {x,y,z}/{roll,pitch,yaw} value shape for the
> Location/Rotation sub-axis branch vs the nested {location:{},rotation:{}} shape
> the Transform branch requires.

The method-name collision itself *is* documented (H3, line 70+); the per-property
**value-object shape** is the part that is not, and is what forced source-reading.

## What to do

**Wiki edit (downstream wiki process — name the page).** On the
`sequencer.add_keyframe` H3 in `docs/wiki-src/sequencer.md`, under the
frame-numbered-form bullet, add a short value-shape table/sub-list:

- `property="Transform"` → nested `{location:{x,y,z}, rotation:{roll,pitch,yaw},
  scale:{x,y,z}}` (subset OK).
- `property="Location"|"Rotation"|"Scale"` → flat `{x,y,z}` (Rotation also
  `{roll,pitch,yaw}`) or `[x,y,z]`.
- numeric/boolean generic property tracks → a bare number / boolean `value`.

A one-line example per shape removes the need to read source. (Optionally mirror
the same note in the auto-generated `RPC_PARAM_OPT` description for `value` in
`SequenceHandler.cpp` so the live `wiki`-against-method schema carries it too.)

**Workaround (today):** for `property="Transform"` nest the value under
`location`/`rotation`/`scale`; for `property="Location"|"Rotation"|"Scale"` pass a
flat `{x,y,z}` (or `{roll,pitch,yaw}` for rotation).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the SEED-mode
  `LogoIntro` cinematic task (seed `sequence.add_keyframe`, 17 calls; judge filed
  the section-range gap). This ticket covers a distinct PROCESS surface from the
  friction note: the `sequencer.add_keyframe` H3 documents the two-shape
  method-name *collision* (seconds vs frame param sets) but never the per-`property`
  **value-object shape**, so the run had to read `SequenceHandler.cpp` to learn
  that `property="Location"`/`"Rotation"` take a flat `{x,y,z}`/`{roll,pitch,yaw}`
  value while `property="Transform"` takes a nested
  `{location:{},rotation:{},scale:{}}` value. Dedup: ripgrep across
  OPEN/IN-REVIEW/DONE/WONTFIX — distinct from
  `E-sequence-add-keyframe-bare-empty-no-success` (the empty `{}` response shape,
  not the input value shape) and from
  `B-sequence-add-keyframe-location-property-rejected` (IN-REVIEW; the per-axis
  *write* fix — this is the *docs* that fix now needs). Confirmed the H3 lacks any
  value-shape detail by reading `docs/wiki-src/sequencer.md:68-76`. Downstream
  wiki target: `docs/wiki-src/sequencer.md` `sequencer.add_keyframe` H3 (add a
  per-property value-shape sub-list with one example each).
