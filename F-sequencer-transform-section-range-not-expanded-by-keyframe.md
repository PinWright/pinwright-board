---
id: F-sequencer-transform-section-range-not-expanded-by-keyframe
title: "Keyframing a transform track leaves its section collapsed; no RPC resizes/expands a section to span its keys"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, add_keyframe, transform-track, section-range]
---

# Keyframing a transform track leaves its section collapsed; no RPC resizes/expands a section to span its keys

Authoring an animation on a transform track via `sequence.add_keyframe`
(`property="Transform"`) writes keys onto the section's double channels but never
grows the section's frame range to cover those keys, and there is no live RPC to
resize an existing section afterward. The result is an authored animation whose
section is collapsed (range `[0,0]`) while its keys sit at frames *outside* that
range — so the push-in the caller built is structurally present (keys exist) but
the section that should play it spans no time.

## Root cause (read in source)

The `property="Transform"` branch of the frame-numbered writer
(`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:2523-2567`–`1659`):

- `Track->FindOrAddSection(0, bSectionAdded)` (`:1567`) creates a fresh
  `UMovieScene3DTransformSection` with its default (collapsed) range.
- Each axis key is written with
  `Channels[N]->GetData().AddKey(TickFrame, FMovieSceneDoubleValue(...))`
  (`:1593`–`:1647`) — this mutates only the channel key data, not the section
  range.
- After `bModified`, the handler calls `MovieScene->Modify()` and returns
  (`:1652`–`:1656`). It **never** calls `Section->SetRange(...)` /
  `Section->ExpandToFrame(TickFrame)` to grow the section to include the new key.

So the section stays at whatever `FindOrAddSection(0)` produced (collapsed),
while keys land at the requested frames (e.g. 0 and 120) outside the section
bounds. Contrast the typed-wrapper / explicit-section paths in the same file
which *do* set a range explicitly (`NewSection->SetRange(...)` at `:1841`,
`FoundSection->SetRange(...)` at `:2577`) — the keyframe writer is the odd one
out.

There is also no live RPC to fix the section after the fact: the sequencer wiki
exposes no `set_section` / `set_section_range` / `resize_section` /
`expand_section` method. `sequencer.list_sections` (DONE) can *read* the
collapsed range but nothing can *write* it; the only section-range writers are
internal to track-creation handlers.

## Evidence (from the audited task)

REALISM-mode cinematic-blockout task (`EstablishingShot`, 36 calls). Friction
note (surface 2 of 2):

> The transform section's range stays collapsed at [0,0] after keyframing (keys
> land at frames 0/120 outside the section bounds) and no RPC in the wiki resizes
> an existing transform section, so the authored push-in section isn't
> auto-expanded to span its own keyframes.

Call sequence: `sequencer.add_transform_track` →
`sequence.add_keyframe {property:"Transform", frame:0, value:{location:{...}}}` (ok) →
`sequence.add_keyframe {property:"Transform", frame:120, value:{location:{...}}}` (ok) →
read-back via `sequencer.list_sections` showed the transform section range
collapsed despite keys at 0 and 120. All calls `ok=true`; the gap is silent —
the writer reports success and the bad range only surfaces on read-back.

## What it should do

Pick one (the first is the natural fix; the second is generally useful on its own):

1. **Auto-expand on keyframe write.** In the `Transform` branch, after the keys
   are added, grow the section to include every written `TickFrame`
   (`Section->ExpandToFrame(TickFrame)` per key, or
   `Section->SetRange(TRange<FFrameNumber>::Hull(Section->GetRange(), TickFrame))`),
   matching how interactive Sequencer auto-sizes a section to its keys. This
   makes the authored animation actually span its keyframes with no extra call.
2. **Add a section-range writer RPC** (e.g. `sequencer.set_section_range` /
   `sequencer.resize_section`, params: section identity by `bindingGuid` +
   `trackName` + `rowIndex`, plus `startFrame`/`endFrame`) so a caller can
   resize any existing section — the write-side counterpart to the read-only
   `sequencer.list_sections` (`F-rpc-sequencer-list-sections`, DONE). This also
   covers non-transform sections and post-hoc trims/extends.

Implementing (1) removes the per-keyframe footgun; (2) closes the broader
read-without-write asymmetry. Doing both is ideal.

**Workaround:** none clean today — the caller cannot resize the collapsed
section through any RPC; re-authoring via a typed wrapper that sets an explicit
range is the only path, and that does not write transform keys.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the
  REALISM cinematic-blockout `EstablishingShot` task (36 calls, outcome
  tool_bug for a *separate* surface). This ticket covers the distinct
  capability gap in the friction note's surface 2: keyframing a transform track
  via `sequence.add_keyframe property="Transform"` leaves the section collapsed
  at `[0,0]` while keys land at frames 0/120 outside it, and no RPC resizes an
  existing section. Root-caused in source: the `Transform` branch
  (`SequenceHandler.cpp:1556`–`1659`) does `FindOrAddSection(0)` + per-channel
  `AddKey(TickFrame, ...)` then only `MovieScene->Modify()` — it never expands
  the section range to the written frames, unlike the explicit-section paths at
  `:1841`/`:2577` that call `SetRange`. No `set_section_range`/`resize_section`
  RPC exists (sequencer wiki has none; `sequencer.list_sections` reads range but
  nothing writes it). Distinct from the judge-filed
  `B-sequence-add-keyframe-location-property-rejected` (which is about
  `property="Location"` being rejected, not the section range). Downstream wiki
  note target if (1) is chosen: `docs/wiki-src/sequencer.md` `add_keyframe` H3.
- `#2-additional-repro` `OPEN` reporter — Additional evidence: independent
  SEED-mode reproduction (seed `sequencer.create`, IntroFlythrough cinematic
  task). Replay-confirmed on a fresh `/Game/OracleCine/OracleFlythrough`
  (24fps, playback 0-120) with a bound camera + transform track:
  `sequence.add_keyframe {property:"Transform", frame:0, value:{location:{x:-800,y:400,z:200},rotation:{roll:0,pitch:-10,yaw:45}}}`
  then the same at `frame:120` (both return bare `{}`), then
  `sequencer.list_sections` returns the transform section with
  `"range":{"start":0,"end":0}` while all six location+rotation channels report
  `"keyCount":2` — keys at 0 and 120 sit outside the collapsed section, so the
  authored dolly spans no time. Confirms the gap reproduces from a different
  task/path and is not asset-specific.
- `#3-additional-repro-per-axis-branch` `OPEN` reporter — Additional evidence:
  SEED-mode reproduction (seed `sequence.add_keyframe`, LogoIntro cinematic
  task) on the **per-axis branch** (`property="Location"`/`"Rotation"`), distinct
  from #1/#2 which both used `property="Transform"`. The per-axis branch
  (`SequenceHandler.cpp` `Location`/`Rotation`/`Scale` block, ~`:1713`–`:1792`)
  shares the same `GetOrAddTransformChannels` scaffolding and the same omission:
  per-channel `AddKey(TickFrame, ...)` + `MovieScene->Modify()` only, no
  `Section->SetRange`/`ExpandToFrame`. Replay-confirmed on a fresh
  `/Game/Cinematics/LogoIntroReplay` with UELogo bound + transform track:
  `sequence.add_keyframe {actorName:"UELogo", property:"Location", frame:0, value:{x:0,y:0,z:0}}`
  then the same at `frame:150 value:{x:0,y:0,z:300}` (both return bare `{}`), then
  `sequencer.list_sections {bindingGuid:C503D63E4FE66CFD03D29B9DAEA66513}`
  returned the transform section with verbatim
  `"range":{"start":0,"end":0}` while the three Location channels each report
  `"keyCount":2` (keys at frames 0 and 150) — section collapsed, keys outside it.
  So the defect is in **both** add_keyframe transform branches, not just
  `property="Transform"`; the auto-expand fix (option 1) must cover the per-axis
  `Location`/`Rotation`/`Scale` writes too.
- `#4-fix-auto-expand` `IN-REVIEW` developer — Implemented **option 1**
  (auto-expand on keyframe write) covering **both** transform branches.
  `SequenceHandler.cpp`: `SequenceHelpers::GetOrAddTransformChannels` now takes an
  optional `UMovieSceneSection** OutSection` out-param and returns the resolved
  `UMovieScene3DTransformSection` (it previously returned channels only); added a
  small `SequenceHelpers::ExpandSectionToFrame(Section, Frame)` helper wrapping
  `Section->ExpandToFrame`. The `property="Transform"` branch and the per-axis
  `Location`/`Rotation`/`Scale` branch each capture the section and, on `bModified`,
  call `ExpandSectionToFrame(TransformSection, TickFrame)` before `MovieScene->Modify()`
  — so the section that `FindOrAddSection(0)` created collapsed at `[0,0]` now grows
  to span every written key (matching interactive Sequencer auto-sizing). Files:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Sequencer/SequenceHandler.cpp`.
  Regression test (new):
  `Source/EditorAutomationRpcGateway/Private/Tests/Sequencer/TestKeyframeExpandsTransformSection.cpp`
  — drives the real registered `sequence.add_keyframe` handler via
  `InvokeHandlerWithCapture` on a real `/Game` LevelSequence with a bound
  possessable, writes a key at frame 120 (`property="Transform"`) and at frame 150
  (`property="Location"`), then asserts the binding's transform section
  `GetRange().Contains(KeyTick)` for the written display-frame→tick. Counterfactual:
  revert the `ExpandToFrame` calls and the section stays `[0,0]`, the contains check
  fails. **Scope:** option 2 (a general `sequencer.set_section_range`/`resize_section`
  writer RPC) is intentionally NOT done here — it is a separable larger feature
  (section-identity params, non-transform sections, post-hoc trims) and should be a
  follow-up F-ticket modeled on the existing `sequencer.set_sub_section_range`
  handler (`SequenceHandler.cpp` `:2660`+).
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule. **Not just a path — the defect is fixed and the mechanics the body names are gone.** `:1556` had drifted into `sequencer.remove_actors`' binding-name resolution. The Transform branch of `sequence.add_keyframe` (registered `:2405`) is `:2523-2567` and now calls `TransformSection->ExpandToFrame(KeyWrite.TickFrame)` at `:2560` under a comment naming the collapsed-`[0,0]` failure this ticket reports; the generic float and bool branches do the same at `:2607` and `:2645`. Keys go through `SequenceKeyframeHelpers::ApplyTransformKeyWrite` (`:2551`), not raw `Channels[N]->GetData().AddKey`, and `Track->FindOrAddSection(0, …)` now lives inside `SequenceHelpers::GetOrAddTransformChannels` (`:330-332`). The ticket's `#4-fix-auto-expand` row landed this; the body was never updated. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
