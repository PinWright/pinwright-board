---
id: E-sequence-add-keyframe-per-axis-value-shape-undocumented
title: "sequence.add_keyframe wiki documents the two call-shape param sets but not the per-axis value-object shape (flat {x,y,z}/{roll,pitch,yaw} vs nested {location,rotation,scale}), forcing source-reading"
status: OPEN
severity: Medium
category: ergonomic
tags: [docs, sequencer, add_keyframe, transform-track, value-shape]
encounters: 6
costly: 4
lastSeen: 2026-07-13T10:19:23.8158526+03:00
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
- `#7-bumped-by-cost` `OPEN` orchestrator — Severity Low -> Medium by cost. Costly encounters counted: #1 (the `LogoIntro` run had to read `SequenceHandler.cpp` to learn the flat vs nested value shape), #4 (the `IntroFlyby` run learned the nested `Transform` shape only by reading `SequenceHandler.cpp`), #5 (the `IntroCutscene` run recovered both the method name and the nested shape by a self-declared last-resort read of `SequenceHandler.cpp:1619`), #6 (the `IntroShowcase` run read both `SequenceHandler`/`SequencerHandler` C++ plus two regression tests to recover the flat shape). Four independent cinematic tasks each paid a plugin-source dive, which is the README's own Medium impact band rather than pure friction. Reach also applies: the encounters span six unrelated cinematic tasks (`LogoIntro`, `IntroMaster`, `CS_Establishing`, `IntroFlyby`, `IntroCutscene`, `IntroShowcase`) and keying a transform is the canonical operation of every one of them.
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
- `#2-additional-decollide-dispatch` `OPEN` reporter — New PROCESS angle from the
  struggle-audit of the SEED-mode `sequencer.delete` cinematic task (focus
  `sequencer.delete`, `/Game/Cinematics/IntroMaster`, 23 calls). Beyond this
  ticket's value-shape doc gap, the **dual registration itself** is an ergonomic
  trap the audited run had to reason around: the `sequencer.add_keyframe` H3
  admits the seconds-based (`SequencerHandler`) and frame-numbered
  (`SequenceHandler`) forms collide under one name and that *"which one wins at
  dispatch time depends on registration order … If you need deterministic
  behavior, prefer the seconds-based form's exact param set; missing required keys
  force the legacy form's path"* — i.e. it tells callers to probe the live schema
  to know which handler they'll hit. The run read the H3, then **defensively chose
  the explicit legacy frame form** (`sequence.add_keyframe` {path,bindingId,
  property,frame,value}) purely to guarantee deterministic dispatch. Friction note
  verbatim: *"add_keyframe is also documented as two colliding dispatch shapes
  under sequencer.add_keyframe, so I deliberately used the explicit legacy
  sequence.add_keyframe name for the transform/Location keys."* Zero failed calls,
  but a reasoning/reading cost every caller pays and a defensive param-shape
  commitment. Proposed fix — **distinct from this ticket's value-shape docs
  remediation**: make dispatch *deterministic* — give the two forms
  non-overlapping method names (or pick one canonical method) so the wiki no
  longer has to say dispatch is registration-order-dependent; if both must
  coexist, document deterministically WHICH args select WHICH form rather than
  "depends on registration order." Downstream wiki page:
  `docs/wiki-src/sequencer.md` `sequencer.add_keyframe` H3. Dedup: this is the
  *collision/dispatch-nondeterminism* angle, distinct from
  `E-sequence-add-keyframe-bare-empty-no-success` (response shape) and from this
  ticket's original *value-object shape* angle; no existing ticket proposes
  de-colliding the two registrations.
- `#3-liveness` `OPEN` reporter — still observed (`CS_Establishing` 24fps 0-5s establishing-shot task, focus `sequencer`, 31 calls). To animate the moving actor's Location on a `MovieScene3DTransformTrack`, the run again had to abandon the modern seconds-based `sequencer.add_keyframe` (documented float-track-only) and fall back to the legacy frame-numbered `sequence.add_keyframe` with `property:"Location"` + a vector value (SAY: "the seconds-based add_keyframe is float-track-only, so I'll use the legacy frame-numbered sequence.add_keyframe writer with property: 'Location' and a vector value"). Same two-keyframe-APIs / two-time-bases juggling for the canonical "make the actor drift" operation; no new angle beyond `#2`'s de-collision proposal. (A candidate feature framing — extend the modern seconds-based `sequencer.add_keyframe` to accept transform/vector channels, or add a dedicated `sequencer.add_transform_keyframe` — is the constructive form of the same split; folded here rather than filed separately since the PROCESS cost is identical to `#2`.)
- `#4-liveness` `OPEN` reporter — still observed (`IntroFlyby` 24fps 0-5s cinematic-flyby task, focus `sequencer.create`, 12 RPCs, zero errors). To key the camera's 5s transform sweep the run again abandoned the float-only modern `sequencer.add_keyframe` and fell back to the legacy frame-numbered `sequence.add_keyframe` with `property:"Transform"` + the nested `{location:{x,y,z}, rotation:{roll,pitch,yaw}}` value — value shape learned only by reading `SequenceHandler.cpp`. Same value-object-shape + method-name-collision + float-only-fallback friction as `#1`–`#3`; no new angle. (The task's separate section-attachment doubt — "do two keyframes land on one section vs split" — is a DISTINCT method/root-cause and was filed as `E-sequencer-add-transform-track-section-model-undocumented`, not folded here.)
- `#5-liveness` `OPEN` reporter — still observed (`IntroCutscene` 24fps film-precision cutscene, focus `sequencer.set_tick_resolution`, 26 logged calls, zero errors). To key the camera dolly the run again abandoned the float-only `sequencer.add_keyframe` and fell back to the legacy frame-numbered `sequence.add_keyframe` with `property:"Transform"` + nested `value:{location:{x,y,z}}` (Location.X 0->800 over frames 0->120) — BOTH the method name (`sequence.` vs `sequencer.`) and the nested value shape were recovered only by a self-declared last-resort read of `SequenceHandler.cpp:1619`. Same method-name-collision + nested-value-shape + float-only-fallback friction as `#1`–`#4`; no new angle.
- `#6-liveness` `OPEN` reporter — still observed (REALISM-mode `IntroShowcase` 24fps 5s establishing-shot cinematic, focus `sequencer`, 19 calls, zero errors). To key the camera's position sweep the run again abandoned the float-only modern `sequencer.add_keyframe` and fell back to the legacy frame-numbered `sequence.add_keyframe` with `property:"Location"` + flat `value:{x,y,z}` (0,0,150 -> 1500,600,150 over frames 0->120) — the WHICH-method (`sequence.` vs `sequencer.`) + flat-value-shape were recovered by a self-declared last-resort read of the plugin `SequenceHandler`/`SequencerHandler` C++ plus two regression tests. Same method-name-collision + value-shape + float-only-fallback friction as `#1`–`#5`; no new angle. (The task's "do both keys land on one auto-expanding section" doubt — settled by reading `TestAddTransformTrackKeyframeReadback.cpp`/`TestKeyframeExpandsTransformSection.cpp` — is the section-model discoverability tracked by `E-sequencer-add-transform-track-section-model-undocumented`, folded there per `#4`, not re-filed.)
