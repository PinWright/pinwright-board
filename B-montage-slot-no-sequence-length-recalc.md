---
id: B-montage-slot-no-sequence-length-recalc
title: "animation.authoring.add_montage_slot appends a slot segment but never recomputes the montage SequenceLength — montage stays duration=0, breaking sections/notifies/playback"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, anim-montage, authoring, sequence-length, duration, silent-noop]
---

# add_montage_slot leaves the montage at SequenceLength=0

The montage authoring chain produces a structurally-broken montage whose
`SequenceLength` (the length `get_animation_info` reports as `duration`) is
**never recomputed** after content is added. `create_montage` makes an empty
montage (length 0, correct), but `add_montage_slot` then appends a real
animation segment to a slot track **without recalculating the montage length**,
so the montage stays `duration=0` even after a multi-second clip is added. Every
downstream read (`get_animation_info`, the `asset.dump` summary, the editor's
montage timeline) reports 0, and section/notify timing has no length to clamp
against.

## Root cause (source-confirmed)

`AnimationAuthoringHandler_Sequence.cpp` `animation.authoring.add_montage_slot`
(handler at :1199) appends the segment and saves, with **no length recompute**:

```cpp
FAnimSegment& Segment = SlotTrack->AnimTrack.AnimSegments.AddDefaulted_GetRef();
Segment.SetAnimReference(Animation);
Segment.StartPos     = StartTime;
Segment.AnimStartTime= 0.0f;
Segment.AnimEndTime  = Animation->GetPlayLength();
Segment.AnimPlayRate = 1.0f;
Segment.LoopingCount = 1;
AnimationAuthoringHelpers::SaveAnimAsset(Montage, bSave);   // <-- length never updated
```

A ripgrep over the whole handler file confirms **zero** calls to
`CalculateSequenceLength` / `SetSequenceLength` / `UpdateLinkableElements` /
`RefreshSegmentOnLoad` anywhere in the Sequence+Montage cluster — nothing
recomputes `UAnimMontage` length after segment edits. The UE montage editor
recomputes length after any slot/segment change (the segment track drives the
montage's overall length); the MCP handler omits this entire step, so the
authored asset's length is stale at 0.

`create_montage` (:1087) is correct to leave length 0 (it adds only an empty
slot track), and `add_montage_section` / `set_section_timing` operate on
`CompositeSections` (section start times), which is a *different* axis from the
slot-segment length — neither recomputes length either, but the slot is where a
real duration should appear, so `add_montage_slot` is the right place to fix.

## Repro (from the audited task, replay-confirmable)

Task: author `MTG_HitReact` (skeleton `SK_Mannequin`, slot `UpperBody`).
1. `create_montage {name:MTG_HitReact, skeletonPath:SK_Mannequin, slotName:UpperBody}` → success.
2. `get_animation_info {assetPath:.../MTG_HitReact}` → `duration: 0` (expected, empty montage).
3. `add_montage_slot {assetPath:..., animationPath:.../MM_HitReact_Front_Lgt_01, slotName:UpperBody}` → `success: true`.
4. `get_animation_info {assetPath:...}` → **`duration: 0` STILL** — the multi-second clip added no length.

The audited agent had to reverse-engineer this from the plugin C++ and UE engine
source, then bypass the verb entirely: `property.set {assetPath, property:"SequenceLength", value:0.7}`
to force a non-zero length. Friction note (verbatim): *"add_montage_slot appends
an FAnimSegment but never recalculates SequenceLength, so the montage stayed
duration=0 (get_animation_info kept reporting 0) — I had to set SequenceLength
via property.set as a workaround … both required reading the plugin C++ handlers
and UE engine source … a notify-time and montage-length discoverability/correctness gap."*

Call counts in the audited task: `get_animation_info` was called **5 times**,
reporting `duration:0` on calls 1-4 and only flipping to `0.7` after the manual
`property.set SequenceLength` workaround — five readbacks burned chasing a length
the build verbs should have produced.

## What it should do

After mutating `SlotTrack->AnimTrack.AnimSegments` in `add_montage_slot`,
recompute and persist the montage length the same way the engine montage editor
does — recalculate the slot/segment-derived length and write it back to the
montage's length field (e.g. `Montage->SetSequenceLength(Montage->CalculateSequenceLength())`
or the 5.7-equivalent length accessor / `UpdateLinkableElements()`), then save.
After the fix, step 4 above should report a non-zero `duration` matching the
added clip's play length, with no `property.set` workaround.

Consider applying the same recompute in the section/composite-segment verbs that
can extend content length so the montage length is always consistent after any
authoring write (the broader pattern is: any verb that edits montage content
must leave length correct, never stale).

This is distinct from the judge-filed `B-add-montage-notify-time-dropped` (that
is the notify `LinkValue`/`TriggerTimeOffset` write/read mismatch — a *notify*
defect). This ticket is the *montage length* defect: a separate field, a
separate handler (`add_montage_slot` vs `add_montage_notify`), a separate read
path (`get_animation_info` duration vs `list_notifies` time). Both surfaced in
the same task and both forced a `property.set` bypass, but they are independent
correctness gaps.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `MTG_HitReact`
  hit-reaction montage authoring task (namespace animation.authoring, outcome
  tool_bug; the notify half filed by the judge as
  `B-add-montage-notify-time-dropped`). Source-confirmed: `add_montage_slot`
  (`AnimationAuthoringHandler_Sequence.cpp:1247-1255`) appends an `FAnimSegment`
  and calls `SaveAnimAsset` with no length recompute; ripgrep over the handler
  file finds zero `CalculateSequenceLength`/`SetSequenceLength`/`UpdateLinkableElements`
  calls in the Sequence+Montage cluster. Live repro: after `create_montage` +
  `add_montage_slot` with a real multi-second clip, `get_animation_info` still
  reports `duration:0` (5 readbacks in the audited task, all 0 until a manual
  `property.set SequenceLength=0.7` forced it). Distinct from the notify-time
  bug (different field/handler/read-path). Fix: recompute and write back the
  montage length after segment edits in `add_montage_slot` (and any other
  content-length-extending montage verb).
- `#2-additional-repro-multislot` `OPEN` reporter — Additional evidence
  (independent replay, different skeleton/asset): authored
  `AM_DinoDragon_IdleToWalk` on `SK_DinoDragon_Skeleton` per a designer
  "idle→walk" montage task. Full chain replayed live via
  `mcp__editor-automation__call`: `create_montage` → two `add_montage_slot`
  (Dino_Idle@0 and Dino_Walk@0.9667, both into `DefaultSlot`) →
  `add_montage_section` Idle@0 + Walk@0.9667 → `link_sections` Idle→Walk →
  `set_blend_in`/`set_blend_out` 0.25. The `asset.dump` `anim_montage.json`
  confirms the slot holds both segments back-to-back (Dino_Walk `startPos`
  0.96666663885116577, so the real montage length is ~1.933s), yet the
  read-back `get_animation_info {assetPath:.../AM_DinoDragon_IdleToWalk}`
  returns verbatim `{"assetType":"AnimMontage","duration":0,"numSections":3,
  "numSlots":2,"numNotifies":0}` — `duration:0` despite two multi-second
  segments. This shows the defect is NOT single-segment-specific (two
  back-to-back segments still yield length 0) and that the section/link/blend
  authoring verbs (`add_montage_section`, `link_sections`, `set_section_timing`,
  `set_blend_in/out`) also never trigger a length recompute — consistent with
  the source finding that nothing in the Sequence+Montage cluster calls
  `CalculateSequenceLength`/`SetSequenceLength`. Surfaced from a seed audit of
  `animation.authoring.link_sections` (link_sections itself round-trips
  correctly — the dump shows Idle.nextSectionName="Walk"); the duration-0
  defect is in `add_montage_slot`, not the seed.
- `#3-fix-recompute-montage-length` `IN-REVIEW` developer — Implemented the
  root-cause fix. In `add_montage_slot`
  (`Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`),
  after appending the `FAnimSegment` to the slot track and before
  `SaveAnimAsset`, added `Montage->SetCompositeLength(Montage->CalculateSequenceLength())`
  — the same montage-editor length-recompute idiom already used by the composite
  verb at `:1754`. NOTE on the ticket's literal Fix text: `SetSequenceLength`
  does not exist and the `SequenceLength` field is deprecated since UE 5.0;
  used the public exported engine API instead (`UAnimMontage::CalculateSequenceLength()`
  takes the max slot-track length, `SetCompositeLength()` writes it back through
  the WITH_EDITOR data-model controller — the ticket's listed "5.7-equivalent
  length accessor" alternative). Also added a `montageLength` field to the
  handler response for readback. Regression test
  `EditorAutomationRpcGateway.animation.authoring.add_montage_slot.RecomputesMontageLength`
  in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAnimationHandlers.cpp`:
  creates a transient skeleton + non-zero-length sequence, runs the real
  `create_montage` + `add_montage_slot` handlers, asserts the empty montage
  starts at duration 0 and gains a non-zero play length approximately equal to
  the added clip after the slot add. Reverting the recompute returns the length
  to 0 and the test fails. (Editor `SetCompositeLength` frame-rounds, so the
  test asserts >0 and within-one-frame, not bit-exact.) Not yet compiled or
  unit-tested — a later phase drives it green.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
