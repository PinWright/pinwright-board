---
id: E-dialogue-context-default-not-reclaimed
title: "audio.authoring.set_dialogue_context never reclaims the engine-seeded empty ContextMappings[0]: append-only, so contextCount is permanently off-by-one, the orphan empty context can't be removed, and no remove/clear verb exists"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [audio, audio-authoring, dialogue, dialogue-wave, set-dialogue-context, orphan-default-context, append-only-orphan-seed, docs]
encounters: 1
lastSeen: 2026-07-11T11:04:29.2655287+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T11:54:00.8914787+03:00
---

# `set_dialogue_context` never reclaims the engine-seeded empty context — contextCount is off-by-one with an unremovable orphan

A freshly created `UDialogueWave` already carries one engine-seeded
`FDialogueContextMapping` with `Speaker:null` / `Targets:[]` — the `UDialogueWave`
constructor always does `ContextMappings.Add(FDialogueContextMapping())`
(engine `DialogueWave.cpp:527`). `audio.authoring.set_dialogue_context` is
**append-only**: it default-constructs a new mapping and adds it, and never
reuses or fills that seeded empty `ContextMappings[0]`. `audio.authoring`
exposes **no** `remove_dialogue_context` / `clear_dialogue_contexts` /
replace-all verb — the five dialogue verbs (`create_dialogue_voice`,
`describe_dialogue_voice`, `create_dialogue_wave`, `describe_dialogue_wave`,
`set_dialogue_context`) are the entire surface.

Consequences for the caller:

- `contextCount` is permanently one higher than the number of mappings you add
  (one `set_dialogue_context` -> `contextCount:2`; two -> `contextCount:3`).
- Every wave keeps a leading orphan `{speaker:"", targets:[]}` context that no
  `set_dialogue_context` created and nothing can delete.
- So a clean create -> wire -> readback round-trip — `contextCount` equal to the
  number of intended mappings, every context carrying an intended speaker — is
  **unreachable through the documented API**. A narrative/localization handoff
  that verifies "contextCount == number of lines I wired" always fails the exact
  check even though every intended speaker/target mapping is present.

This is the **Dialogue member of an established orphan-default-seed family** —
same shape, same "append-only verb never reclaims the engine seed, no remove
verb" root cause, as:
- `E-create-struct-orphan-seed-field` — `blueprint.create_struct` seeds a default
  `MemberVar_0`; `add_struct_field` never reclaims it (fixed by reclaim-on-first-add).
- `B-create-montage-duplicate-default-slot` — `create_montage` leaves a duplicate
  empty `DefaultSlot` with no `remove_montage_slot` verb.

## What it should do

Pick one (the first mirrors the sibling reclaim fixes and is the clean ergonomic
route; the second is the doc floor):

1. **Reclaim the seed on first set.** On the FIRST `set_dialogue_context` for a
   wave, populate the engine-seeded empty `ContextMappings[0]` (if it is the
   pristine default — `Speaker:null`, `Targets:[]`) instead of appending a new
   mapping — the same find-or-reuse idiom `E-create-struct-orphan-seed-field`'s
   fix and `add_montage_slot` already use. Then `contextCount` equals the number
   of intended mappings and no orphan remains. (Optionally also add a
   `remove_dialogue_context` / `clear_dialogue_contexts` verb so a caller can
   prune an unwanted context after the fact.)

2. **At minimum, document it.** In `docs/wiki-src/audio.authoring.md`
   (`set_dialogue_context` and `create_dialogue_wave` sections), note that a fresh
   `DialogueWave` carries an engine-seeded empty context, so `contextCount` /
   `describe_dialogue_wave` will report one more context than the mappings you
   added and will show a leading `{speaker:"", targets:[]}` entry — and that there
   is no verb to remove it.

**Workaround:** rewrite `ContextMappings` via `property.set` to drop the leading
empty entry (there is no typed remove verb), or simply accept the orphan empty
context and count `contextCount - 1`.

## Evidence (this task's call-log + friction)

Struggle-audit of a two-character dialogue build (focus
`audio.authoring.describe_dialogue_wave`; outcome `blocked_by_tool`). Method
discovery was smooth (index + one grep + the method pages). The friction was
entirely the surprising off-by-one:

- One `set_dialogue_context` on the fresh `DW_Reply` (companion -> [hero])
  returned `contextCount:2`; final readback `contextCount:2` for one intended
  mapping.
- `DW_Greeting` after two `set_dialogue_context` calls (hero -> [companion],
  companion -> [hero]) read back `contextCount:3`.
- `describe_dialogue_wave` (this run's focus, which behaved correctly and
  faithfully surfaced the state) showed the persistent leading
  `{speaker:"", targets:[]}` orphan on every wave.

To explain a single call returning `contextCount:2`, the agent had to leave the
wiki and read source: engine `DialogueTypes.h` + `DialogueWave.cpp` (found the
ctor `ContextMappings.Add()` at :527) and, as a last resort, the plugin's
`set_dialogue_context` handler (`AudioAuthoringHandler.cpp`) to confirm the
append-only-no-reclaim behavior; an index-wide search of `audio.authoring.md`
found no remove/clear/replace-all context helper. Friction note (verbatim):

> Struggled: a single set_dialogue_context returned contextCount 2, which was
> confusing; I read UE engine source (DialogueWave.cpp:527 default context,
> FDialogueContext null-target default) and, as a last resort, the plugin's
> set_dialogue_context C++ to confirm the append-only + stray-null-target
> behavior; index-wide search of audio.authoring found no one-shot
> remove/clear/replace-all context helper.

## Distinct from existing dialogue tickets

- **`B-dialogue-context-null-target-prepended`** (IN-REVIEW) is the stray `null`
  prepended INSIDE a written mapping's `Targets` array — a different defect with a
  different fix (`Targets.Reset()` in the handler). That ticket **explicitly
  scopes this off-by-one OUT** ("A freshly-created DialogueWave already carries one
  default FDialogueContextMapping ... That default-empty mapping is engine-side and
  is NOT the subject of this ticket"). This ticket is that carved-out,
  otherwise-untracked issue: the unremovable engine-seeded empty context and the
  append-only verb / missing remove verb.
- **`E-audio-dialogue-no-readback-get-info-unknown`** (IN-REVIEW) is about the
  absence of a dialogue readback (`get_audio_info` -> `type:"Unknown"`), addressed
  by the now-shipped `describe_dialogue_*` verbs. This ticket is not a readback gap
  — the readback works; the write surface can't reach a clean context list.

severity rationale: impact=soft-blocker (clean context-list round-trip unreachable
via documented API; workaround = `property.set` rewrite or count-minus-one) x
reach=first-class dialogue authoring family but not every-session -> Medium.

## History
- `#2-fix` `IN-REVIEW` developer — GO. Reclaim-on-first-set fix shipped. Root cause (re-verified against source): `set_dialogue_context`'s non-replace path unconditionally `Wave->ContextMappings.Add(NewMapping)` and the replace path matches only by `Speaker`, so the null-speaker engine seed (`UDialogueWave` ctor `ContextMappings.Add(FDialogueContextMapping())`, `DialogueWave.cpp:527`) was never reclaimed → `contextCount` permanently one higher than the contexts authored. Fix in `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp` (`set_dialogue_context`): added an `AddOrReclaimSeed` helper called from BOTH the replace-not-found and non-replace branches — on the first authored set, if the wave holds exactly one pristine seed (`Speaker==null && SoundWave==null && Targets empty-or-lone-null && LocalizationKeyFormat=="{ContextHash}"`) it reuses `ContextMappings[0]` in place instead of appending; once a real context exists the slot is no longer pristine and later sets append as before. The empty-or-lone-null guard is required because the seed's `FDialogueContext` ctor does `Targets.AddZeroed()` (`DialogueTypes.cpp:23`), so the in-memory pristine `Targets` is `[null]`, not `[]`. Mirrors `E-create-struct-orphan-seed-field`'s `IsUntouchedSeedField` strict-guard reclaim. Regression test (adopted the red test): `PinWright.audio.authoring.set_dialogue_context.FirstSetReclaimsSeedMapping` in `Source/PinWright/Private/Tests/Media/TestDialogueContextReclaim.cpp` — observed FAILING pre-fix (`Num()==2` / `contextCount:2`), now PASSES post-fix (`Num()==1`, `contextCount:1`, the single reclaimed mapping carries the speaker set). Plugin compiled clean; scoped test run `Result={Success}`. Not a duplicate of the co-located `B-dialogue-context-null-target-prepended` (distinct stray-null-inside-Targets bug; E does not depend on it and did not touch it). Scope split: the optional `remove_dialogue_context`/`clear_dialogue_contexts` verb is carved out to new ticket `F-audio-dialogue-remove-clear-context-verb` (net-new API surface); the docs-floor alternative (Fix #2) is superseded by the reclaim, not shipped.
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `audio.authoring` two-character dialogue build (focus `audio.authoring.describe_dialogue_wave`, outcome `blocked_by_tool`; the stray-null-INSIDE-Targets defect is filed separately by the judge as `B-dialogue-context-null-target-prepended`, which explicitly scopes THIS off-by-one out). Distinct PROCESS angle: the engine-seeded empty `ContextMappings[0]` (`UDialogueWave` ctor `ContextMappings.Add()`, `DialogueWave.cpp:527`) is never reclaimed by the append-only `set_dialogue_context`, and `audio.authoring` has no `remove_dialogue_context`/`clear`/replace-all verb, so `contextCount` is permanently off-by-one (1 mapping -> 2, 2 -> 3) with an unremovable orphan `{speaker:"",targets:[]}` context and a clean round-trip is unreachable. Call-log: `DW_Reply` `contextCount:2` after one mapping, `DW_Greeting` `contextCount:3` after two; `describe_dialogue_wave` (focus, behaved correctly) surfaced the orphan on every wave; the agent had to read engine `DialogueWave.cpp:527` + plugin `AudioAuthoringHandler.cpp` `set_dialogue_context` and index-search `audio.authoring.md` (no remove verb found) to understand it. Same orphan-default-seed shape as `E-create-struct-orphan-seed-field` and `B-create-montage-duplicate-default-slot`. Fix: reclaim the pristine seeded `ContextMappings[0]` on the first `set_dialogue_context` (mirror the struct/montage reclaim idiom) and/or add a `remove_dialogue_context` verb; at minimum document the seeded empty context + off-by-one in `docs/wiki-src/audio.authoring.md` (`set_dialogue_context` / `create_dialogue_wave` sections).
