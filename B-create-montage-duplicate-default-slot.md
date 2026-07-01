---
id: B-create-montage-duplicate-default-slot
title: "animation.authoring.create_montage produces TWO slot tracks both named 'DefaultSlot' (factory seeds one, handler unconditionally adds another) — doc promises 'one slot', asset is left with a duplicate-named empty slot and no remove verb"
status: IN-REVIEW
severity: Medium
category: bug
tags: [animation, anim-montage, authoring, create-montage, slot, duplicate-slot, silent-malformed]
---

# create_montage leaves a montage with two identically-named "DefaultSlot" tracks

`animation.authoring.create_montage` is documented (handler description and wiki
`create_montage.md`) to create a montage **"with one slot and an empty 'Default'
section"**. In fact it creates **two** slot tracks, both named `DefaultSlot`,
immediately on creation — before any `add_montage_slot` call. A montage slot is
addressed by name (AnimGraph slot nodes resolve by `SlotName`), so two slots
sharing one name is an ambiguous/malformed asset: the duplicate is empty clutter,
it inflates `numSlots` to 2, and there is **no `remove_montage_slot` verb** to
clean it up (the only montage-slot verb is `add_montage_slot`). The agent in the
audited task was left with a populated `DefaultSlot` plus a vestigial empty
`DefaultSlot` it could not remove without dropping to `property.set`.

## Root cause (source-confirmed)

`AnimationAuthoringHandler_Sequence.cpp` `animation.authoring.create_montage`
(handler at :1087). The `UAnimMontage` creation path seeds a default slot track
(UE's montage construction already provides a `DefaultSlot`), and then the handler
**unconditionally adds a second** at :1138-1142:

```cpp
// Add default slot
if (!SlotName.IsEmpty())
{
    FSlotAnimationTrack& SlotTrack = NewMontage->SlotAnimTracks.AddDefaulted_GetRef();
    SlotTrack.SlotName = FName(*SlotName);   // SlotName defaults to "DefaultSlot"
}
```

`AddDefaulted_GetRef()` here is not guarded by a "does a slot of this name
already exist?" check — so the factory-seeded `DefaultSlot` plus this added
`DefaultSlot` make two. Contrast `add_montage_slot` (:1230-1244), which does the
correct **find-or-create** loop over `SlotAnimTracks` before adding, so it never
duplicates. `create_montage`'s slot-add is simply missing that same idempotency
guard.

## Repro (replay-confirmed live via mcp__editor-automation__call)

Deterministic — reproduced on two freshly-created montages.

1. `create_montage {name:"AM_ReplayTest_BiteCombo", path:"/Game/Authored/ReplayTest",
   skeletonPath:"/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton",
   slotName:"DefaultSlot", save:false}` → `success:true`.
2. `get_animation_info {assetPath:".../AM_ReplayTest_BiteCombo"}` →
   `{"assetType":"AnimMontage","duration":0,"numSections":1,"numSlots":2,"numNotifies":0}`
   — **numSlots:2 with zero add_montage_slot calls yet.**
3. `asset.dump {assetPath:".../AM_ReplayTest_BiteCombo"}` → the `anim_montage.json`
   sidecar shows two slots, both named `DefaultSlot`, both with empty `segments`:

   ```json
   "slots": [
     { "segments": [], "slotName": "DefaultSlot" },
     { "segments": [], "slotName": "DefaultSlot" }
   ]
   ```

4. A second `create_montage` (`AM_ReplayTest2`, no explicit slotName) also reports
   `numSlots:2` immediately — confirming the duplicate comes from the factory +
   handler combination, not from caller input.
5. After `add_montage_slot {animationPath:.../Bite_ReplaySrc, slotName:"DefaultSlot"}`
   the dump shows the **first** `DefaultSlot` populated with the segment and the
   **second** `DefaultSlot` still empty — proving `add_montage_slot`'s find-or-create
   reused the first slot (no third slot created) and that the spare empty slot is a
   pre-existing `create_montage` artifact, not an `add_montage_slot` one.

The audited agent's friction note (verbatim): *"create_montage AND the factory each
create a 'DefaultSlot', so after add_montage_slot the asset has a populated DefaultSlot
plus an empty duplicate DefaultSlot with no remove verb available — left as benign
clutter."*

## What it should do

`create_montage` should leave **exactly one** slot of the requested name (the doc's
"one slot"). Fix: in the create_montage handler (:1138-1142), guard the slot add
with the same find-or-create check `add_montage_slot` already uses — only
`AddDefaulted_GetRef()` a new `FSlotAnimationTrack` if no existing
`SlotAnimTracks[i].SlotName == FName(*SlotName)`; otherwise reuse / rename the
factory-seeded slot. (Equivalently: if the factory always seeds exactly one
`DefaultSlot`, rename that seeded track to `SlotName` instead of appending a new
one.) After the fix, step 2 should report `numSlots:1` and the dump should show a
single slot.

This is distinct from the existing montage tickets:
- `B-montage-slot-no-sequence-length-recalc` — the montage `duration:0` length
  bug (a *length* field on `add_montage_slot`, not slot count on `create_montage`).
- `E-get-animation-info-thin-on-montage` — `get_animation_info` reports only counts
  (a readback-parity gap; here the count itself is *wrong* because the slot is
  genuinely duplicated, which is a write-side defect, not a readback thinness).
- The "vestigial Default *section*" the same agent also hit is **documented**
  behavior ("an empty 'Default' section" in the description) — not this bug; this
  ticket is the duplicate *slot* only, which the doc explicitly contradicts ("one
  slot").

**Workaround:** rewrite `SlotAnimTracks` (or the dumped `slots[]`) via
`property.set` to drop the duplicate empty slot; there is no typed remove verb.

## History
- `#1-initial-repro` `OPEN` reporter — Realism audit of a DinoDragon bite-combo
  montage authoring task (namespace animation.authoring, outcome tool_bug, no seed
  / realism mode; culprit `animation.authoring.create_montage`). Replay-confirmed
  live via `mcp__editor-automation__call` on `SK_DinoDragon_Skeleton`:
  `create_montage` immediately yields `numSlots:2` (before any `add_montage_slot`),
  and `asset.dump`'s `anim_montage.json` shows two slots both named `DefaultSlot`
  with empty `segments`; reproduced on a second montage. Source-confirmed root
  cause: `AnimationAuthoringHandler_Sequence.cpp:1138-1142` unconditionally
  `AddDefaulted_GetRef()`s a `DefaultSlot` on top of the factory-seeded one, with
  no find-or-create guard (unlike `add_montage_slot` at :1230-1244 which does
  guard). Contradicts the documented "one slot" contract; leaves a duplicate-named
  empty slot with no `remove_montage_slot` verb to clean it. Distinct from the
  length bug (`B-montage-slot-no-sequence-length-recalc`), the readback-thinness
  ergo (`E-get-animation-info-thin-on-montage`), and the documented "Default
  section" behavior.
- `#2-fix` `IN-REVIEW` developer — Replaced the unconditional slot append in the
  `create_montage` handler with a find-or-rename guard so the asset ends with
  exactly one slot of the requested name. The `UAnimMontage` constructor (via
  `UAnimMontageFactory`) already seeds one `DefaultSlot`; the handler now (a) reuses
  a track already named `slotName`, else (b) renames the lone seeded empty slot
  (the default `slotName="DefaultSlot"` path), else (c) appends a new track —
  mirroring `add_montage_slot`'s find-or-create idiom (:1230-1244). This also fixes
  the non-default-`slotName` case the ticket flagged (a bare find-or-create would
  have left the seeded `DefaultSlot` plus the requested slot = 2; the rename yields
  a single slot in all cases). File:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`
  (create_montage handler, formerly :1138-1142). Regression test added:
  `FAuthoringCreateMontageSingleSlotTest`
  (`EditorAutomationRpcGateway.animation.authoring.create_montage.SingleSlotNoDuplicate`)
  in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAnimationHandlers.cpp`
  — drives the real handler twice (default and custom `slotName`) and asserts
  `SlotAnimTracks.Num() == 1` with the requested name each time; reverting the fix
  makes both montages report two `DefaultSlot` tracks and the assertions fail. Not
  compiled here (a later phase compiles + runs).
