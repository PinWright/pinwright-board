---
id: F-montage-section-remove-rename
title: "No remove/rename verb for an AnimMontage CompositeSection — the create_montage-seeded 'Default' section can't be cleaned up without dropping to property.get/property.set on CompositeSections"
status: WONTFIX
severity: Medium
category: feature
tags: [animation, anim-montage, authoring, montage-section, lifecycle, property-set-fallback]
---

# No remove/rename verb for a montage section

`create_montage` (handler description, `AnimationAuthoringHandler_Sequence.cpp:1088`)
documents that it seeds the new `UAnimMontage` with **"an empty 'Default' section"**,
expecting the caller to "build out sections … via `animation.authoring.add_montage_section`".
The montage-section surface is **append-and-edit only**:

- `animation.authoring.add_montage_section` (`:1157`) — append a `FCompositeSection`.
- `animation.authoring.set_section_timing` (`:1276`) — edit an existing section's time.
- `animation.authoring.link_sections` — set `nextSectionName`.

There is **no `remove_montage_section` and no `rename_montage_section`**
(`REGISTER_RPC_HANDLER` grep over `Source/` finds only the three above). So once the
factory has seeded the `Default` section, a caller who wants the montage to contain
*exactly* their own sections (here Bite1/Bite2) has no typed way to drop or rename the
vestigial `Default`: after `add_montage_section ×2` the asset carries **3** sections
(`Default` + Bite1 + Bite2), and no verb removes the first. This is the same
missing-remove-verb shape already filed for the duplicate slot
(`B-create-montage-duplicate-default-slot`, which notes "no `remove_montage_slot`
verb") — but for **sections**, and unlike the slot duplicate (a bug the doc
contradicts) the leftover `Default` *section* is **documented** behavior, so it is a
feature gap (a needed lifecycle verb), not a bug. It is the section analogue the
slot ticket explicitly carved out of scope.

## Process friction (this is the PROCESS angle — outcome already judged)

The audited DinoDragon bite-combo task hit this as concrete extra work. The typed
authoring calls all succeeded, but to satisfy "exactly Bite1/Bite2" the agent had to
abandon the typed surface and drop to generic reflection — **4 extra calls** that a
single `remove_montage_section` would have collapsed to one:

1. `asset.dump` the montage → `anim_montage.json` (to see the leftover `Default`).
2. `property.get CompositeSections` (to capture the faithful struct values to preserve).
3. `property.set CompositeSections` (rewrite the array dropping `Default`, keep Bite1+Bite2).
4. `get_animation_info` (re-confirm `numSections=2`).

Agent friction note (verbatim): *"create_montage's factory seeds a vestigial 'Default'
section that no typed verb can remove or rename, leaving 3 sections; the task wants
exactly Bite1/Bite2, so I fell back to property.get/property.set to rewrite the
CompositeSections array (noted as a last-resort reflection workaround)."*

Rewriting a `TArray<FCompositeSection>` through `property.set` is exactly the
hand-serialize-the-whole-struct-array fallback the typed authoring surface exists to
avoid: it forces the agent to read every existing field back, re-emit the struct
array verbatim minus one element, and hope nothing is dropped — brittle and
error-prone for a one-line intent ("there should be no Default section"). Same class
as `F-niagara-remove-emitter` (a missing remove verb whose only workaround was
property/reflection element-removal that the generic path doesn't support cleanly)
and `F-anim-composite-asset-creator` (which deferred segment-remove until 2+ callers
— this is a concrete caller for the section equivalent).

## What it should do

Add two symmetrical lifecycle verbs mirroring `add_montage_section` /
`set_section_timing`:

1. `animation.authoring.remove_montage_section(assetPath, sectionName, save?)` —
   remove the named `FCompositeSection` from `UAnimMontage::CompositeSections`,
   repairing any `nextSectionName` links that pointed at it (clear or relink), and
   return the new section count. Resolves the seeded-`Default` cleanup directly.
2. `animation.authoring.rename_montage_section(assetPath, sectionName, newName, save?)`
   — rename in place, updating any `nextSectionName` references. (Renaming the seeded
   `Default` to the caller's first real section name is an alternative one-call fix to
   the same problem.)

Implementation surface: `UAnimMontage::CompositeSections` is the same
`TArray<FCompositeSection>` the existing `add_montage_section` / `set_section_timing`
handlers already mutate; the dump builder (`AnimMontageDumpBuilder.cpp`) already reads
`SectionName`/`NextSectionName`/`GetTime()`, so the readback shape is known. Reuse
`SaveAnimAsset()` for the `save?` path. Surface a `SECTION_NOT_FOUND` error matching
the existing montage error vocabulary.

**Workaround (current):** `asset.dump` → `property.get CompositeSections` →
`property.set CompositeSections` (hand-rewrite the struct array dropping the unwanted
section) → `get_animation_info` to re-confirm. 4 calls, brittle struct-array
round-trip.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  `AM_DinoDragon_BiteCombo` bite-combo authoring task (namespace
  animation.authoring, outcome tool_bug; the duplicate-*slot* half is the
  judge-filed `B-create-montage-duplicate-default-slot`). This ticket is the
  distinct *section*-lifecycle process angle: `create_montage` seeds a `Default`
  section (documented at `AnimationAuthoringHandler_Sequence.cpp:1088`) that no
  typed verb removes or renames — `REGISTER_RPC_HANDLER` grep over `Source/` finds
  only `add_montage_section` (:1157), `set_section_timing` (:1276), and
  `link_sections` for sections, no remove/rename. The agent had to drop to
  `asset.dump` → `property.get`/`property.set` on `CompositeSections` (4 extra calls)
  to rewrite the section array down to exactly Bite1/Bite2, a brittle
  whole-struct-array reflection round-trip. Friction note quoted verbatim above.
  Proposes `remove_montage_section(assetPath, sectionName, save?)` +
  `rename_montage_section(assetPath, sectionName, newName, save?)` over
  `UAnimMontage::CompositeSections`, repairing `nextSectionName` links, mirroring the
  existing add/set handlers and reusing `SaveAnimAsset()`. Same missing-remove-verb
  class as `B-create-montage-duplicate-default-slot` (slots, "no remove_montage_slot")
  and `F-niagara-remove-emitter`; the slot ticket explicitly leaves the section case
  out of scope (it is documented behavior → feature, not bug).
- `#2-retriage` `OPEN` triage — Low→Medium: missing remove/rename verb forces a brittle whole-struct-array property.set round-trip with silent-field-loss risk; niche montage path.
- `#3-wontfix` `WONTFIX` developer — The Medium severity rests on a false workaround premise. The ticket asserts the only fallback is a brittle 4-call `asset.dump` → `property.get`/`property.set CompositeSections` whole-struct-array round-trip "with silent-field-loss risk" (the #2-retriage Low→Medium rationale). That is wrong: the generic `container.array.remove` handler (`UtilityPropertyHandler.cpp:1944`) removes an array element **by index** via `FScriptArrayHelper::RemoveValues(Index, 1)` (`:1970`), resolving the asset through the same `RESOLVE_ARRAY_PROPERTY` reflection resolver `property.get`/`property.set` use — which the ticket itself confirms works on `CompositeSections`. Index removal preserves every other element's bytes verbatim, so there is no struct re-serialization and no field-loss risk. Dropping the seeded `Default` is **one** call: `container.array.remove {objectPath:<montage>, propertyName:"CompositeSections", index:0}`. The reported "exactly Bite1/Bite2" friction is one index:0 removal away. The seeded `Default` is the head section (nothing's `NextSectionName` points *at* it — `create_montage` seeds it at index 0 and never links to it, `AnimationAuthoringHandler_Sequence.cpp:1095-1165`), so even the link-repair concern does not arise in the reported case. The cited `F-niagara-remove-emitter` precedent (DONE, Critical) argues AGAINST this ticket: it was justified precisely because the generic reflection path could NOT remove `TArray<FNiagaraEmitterHandle>` elements (genuinely no workaround) — here generic array-element removal DOES work on `CompositeSections`. The only genuine residual (a forward `NextSectionName` pointing at a removed/renamed mid-chain section left dangling) is a narrow edge that does not occur in the reported scenario and is a Low-tier ergonomic gap, not a Medium two-verb + link-repair feature. WONTFIX: a working one-call generic workaround exists; the severity rationale collapses.
