---
id: E-audio-get-info-soundclass-mix-readback-thin
title: "audio.authoring has no live readback for SoundClass voiceCenterChannelVolume/ChildClasses or any SoundMix per-adjuster value (get_audio_info gives only modifierCount) — needs describe_sound_class / describe_sound_mix to match the describe_attenuation sibling pattern"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [audio, soundclass, soundmix, readback, describe, inspect-after-mutate, wiki]
---

# `audio.authoring` lacks a live readback for SoundClass leaf properties and SoundMix adjusters

This is the **SoundClass / SoundMix sibling** of
`E-audio-authoring-attenuation-readback-undocumented` (IN-REVIEW), the same
readback-thinness shape scoped to SoundAttenuation. That sibling was NOT closed
with a docs-only `asset.dump` fallback note — it shipped a live
`describe_attenuation` reader (`AudioAuthoringHandler.cpp:2518`) plus an overlay
line, and the `## Inspect-after-mutate` section was rewritten to route
verification through the `describe_*` family
(`docs/wiki-src/audio.authoring.md:5-7`). SoundClass and SoundMix are now the
ONLY two authoring types in that section with no `describe_*` reader, so an agent
that mutates them with `set_class_properties` / `add_mix_modifier` /
`create_sound_mix` has no in-namespace live way to confirm the written values and
must leave for an `asset.dump` -> `properties.json` (or `property.get`) pivot.

The thinness is concrete in `get_audio_info`
(`AudioAuthoringHandler.cpp:2633-2649`):

- **SoundClass** (`get_audio_info`, lines 2633-2644) emits only `volume`,
  `pitch`, `parentClass`, and `outputSubmix`. It never emits
  **`voiceCenterChannelVolume`** — even though `set_class_properties` writes that
  exact field (`AudioAuthoringHandler.cpp:1361`,
  `SoundClass->Properties.VoiceCenterChannelVolume = ...`; it also writes
  `lowPassFilterFrequency` / `lfeBleed`, equally unread). Nor the parent-side
  `ChildClasses` link that `set_class_parent`/`create_sound_class` now maintain
  (`SetSoundClassParentMaintainingChildren`, :212) — `B-sound-class-parent-no-child-link`
  (IN-REVIEW) explicitly deferred the `childClasses` readback widening to THIS
  ticket: it "independently owns the get_audio_info widening; coordinate there to
  avoid doing it twice."
- **SoundMix** (`get_audio_info`, lines 2645-2649) emits only **`modifierCount`**
  (`Mix->SoundClassEffects.Num()`). It never emits the per-adjuster
  `soundClass` / `volumeAdjuster` / `pitchAdjuster` / `applyToChildren` values
  that `add_mix_modifier` (:1576-1582) and `create_sound_mix` `classAdjusters`
  (:1511-1531) write. So an agent that adds two modifiers can confirm *that there
  are two* but not *what they are*.

So `audio.authoring` can write a SoundClass leaf-property tune and a SoundMix
adjuster ladder but offers no live reader that confirms either — the exact
"documented mutate verb, no matching live read" gap the `describe_attenuation`
sibling already closed for SoundAttenuation.

## Evidence (this task)

Seed `audio.authoring.set_class_parent`; the story built a full SoundClass
mixing tree (Master -> {Gameplay -> SFX, Music, Dialogue}) plus a `DefaultMix`
SoundMix with two modifiers, then asked to "read back ... so I can confirm the
parent/child tree and the volume edits landed correctly." The build itself ran
clean — all five `get_audio_info` readbacks succeeded — but two of the requested
confirmations weren't in the payload, and the agent had to pivot. Friction note
(verbatim):

> "Minor: get_audio_info doesn't surface voiceCenterChannelVolume or
> per-adjuster values (only modifierCount), so I fell back to asset.dump
> properties.json sidecars to verify those; and asset.dump initially rejected
> param 'path' (needed 'assetPath'), one retry each."

The call log shows the recovery cost concretely: after the seven intended audio
calls (including five `get_audio_info` readbacks and one `get_audio_info` on
DefaultMix that returned `modifierCount 2`), the agent emitted four extra calls
to recover the missing fields — `call("asset")` (namespace index nav),
`call("asset.dump")` (wiki-nav), then two `asset.dump` executes (Dialogue
properties, DefaultMix properties). The first two `asset.dump` attempts also ate
the `path` vs `assetPath` drift (see
`E-asset-path-vs-assetpath-list-drift #3`), compounding the pivot. The
`voiceCenterChannelVolume` confirm and the modifier-value confirm were *both*
named in the story's step 5/6 and step 7 readback, so this is squarely the
documented-readback-can't-confirm-the-write shape.

## Fix

Ship two live readers mirroring `describe_attenuation`
(`AudioAuthoringHandler.cpp:2518`), so the fields the mutate verbs write are
confirmable in-namespace without an `asset.dump`/`property.get` pivot:

- **`audio.authoring.describe_sound_class { assetPath }`** — echo the full
  `FSoundClassProperties` surface `set_class_properties` writes: `volume`,
  `pitch`, `lowPassFilterFrequency`, `lfeBleed`, **`voiceCenterChannelVolume`**,
  plus `parentClass`, the submix (`outputSubmix`, MCP_HAS_SUBMIX-gated like
  `get_audio_info`), and a **`childClasses`** array of the parent-side links that
  `SetSoundClassParentMaintainingChildren` maintains (closing the readback half
  `B-sound-class-parent-no-child-link` deferred here).
- **`audio.authoring.describe_sound_mix { assetPath }`** — echo `modifierCount`
  plus an **`adjusters`** array, one object per `Mix->SoundClassEffects` entry
  with `soundClass` / `volumeAdjuster` / `pitchAdjuster` / `applyToChildren` (the
  values `add_mix_modifier` and `create_sound_mix` `classAdjusters` write), plus
  the mix-level `applyEQ` / `eqPriority` / `fadeInTime` / `fadeOutTime` that
  `add_mix_modifier` sets. Echo only fields the model actually stores —
  `FSoundClassAdjuster` has NO per-adjuster fade member (engine `SoundMix.h`), so
  do not invent per-adjuster fade keys (`B-add-mix-modifier-fade-params-dropped`
  is the separate write-side fade-drop bug).

Then add an overlay note under `## Inspect-after-mutate` in
`docs/wiki-src/audio.authoring.md` naming both new readers alongside
`describe_attenuation`, so the verify path for SoundClass/SoundMix edits stays in
the namespace. This is the same describe-verb pattern the SoundAttenuation
sibling shipped — NOT the interim `asset.dump`/`property.get` fallback an earlier
draft of this ticket proposed (that fallback contradicts the overlay's current
"confirm what the verbs wrote without falling back to asset.dump" guidance at
`audio.authoring.md:77`).

**Workaround (until the describe verbs land):** confirm SoundClass
`voiceCenterChannelVolume` / `childClasses` and SoundMix per-adjuster values via
`asset.dump { assetPath }` -> `properties.json`, or `property.get` on the mix's
`SoundClassEffects` array (note: the array UPROPERTY is `SoundClassEffects`, not
`SoundClasses` — see `#4`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `audio.authoring`
  SoundClass-hierarchy + SoundMix build (seed `audio.authoring.set_class_parent`,
  outcome done; the parent/child wiring bug filed separately by the judge as
  `B-sound-class-parent-no-child-link`). Distinct PROCESS angle: the
  `audio.authoring` overlay's `## Inspect-after-mutate` section points agents at
  `get_audio_info` to verify writes, but the handler
  (`AudioAuthoringHandler.cpp:2420-2436`) omits SoundClass
  `voiceCenterChannelVolume` (written at :1243-1244) and emits only
  `modifierCount` for a SoundMix, never the per-adjuster values — so confirming a
  leaf-property tune or a mix modifier forces an `asset.dump` ->
  `properties.json` pivot the overlay never documents. Friction note (verbatim):
  "get_audio_info doesn't surface voiceCenterChannelVolume or per-adjuster values
  (only modifierCount), so I fell back to asset.dump properties.json sidecars to
  verify those." Call log confirms 4 recovery calls (`asset` index nav,
  `asset.dump` wiki-nav, two `asset.dump` executes); the `asset.dump` pivot
  additionally tripped the `path`/`assetPath` drift (appended as
  `E-asset-path-vs-assetpath-list-drift #3`). Same readback-thinness shape as
  the SoundAttenuation sibling `E-audio-authoring-attenuation-readback-undocumented`,
  here on SoundClass + SoundMix. Proposed: add a line to
  `docs/wiki-src/audio.authoring.md` `## Inspect-after-mutate` naming the
  `get_audio_info` field gaps and pointing at `asset.dump` -> `properties.json`
  for the omitted fields; optionally widen `get_audio_info` at the source.
- `#2-additional-soundmix-adjuster` `OPEN` reporter — Additional evidence
  (REALISM-mode SoundClass-hierarchy + combat-duck SoundMix build): independent
  live replay reproduces the SoundMix half exactly. After
  `create_sound_mix { SM_CombatDuck, classAdjusters: [{SC_Music, 0.4}, {SC_Dialogue,
  1.0}] }`, `get_audio_info { /Game/Audio/Mix/SM_CombatDuck }` returns verbatim
  `{"assetClass":"SoundMix","type":"SoundMix","modifierCount":2,"message":"Audio
  info retrieved"}` — `modifierCount:2` only, no per-adjuster `soundClass` /
  `volumeAdjuster` / `pitchAdjuster`, so the requested confirm "Music ducked to
  0.4, Dialogue held at 1.0" cannot be read back through the documented verifier.
  The root SoundClass is equally thin: `get_audio_info { SC_Master }` returns
  `{"type":"SoundClass","volume":1,"pitch":1,...}` — no `bApplyEQ`/EQ state and no
  child-side class list, so the master's EQ and hierarchy can't be confirmed from
  the reader either. The attempt agent recovered via `property.get` on
  `SM_CombatDuck.SoundClassEffects` and `.bApplyEQ` (a different fallback than the
  `#1` `asset.dump` pivot, but the same leave-the-reader-to-confirm-the-write
  cost). Confirms the handler is unchanged at the same site (now
  `AudioAuthoringHandler.cpp:2527-2531` SoundMix `modifierCount`-only;
  :2515-2526 SoundClass omits `voiceCenterChannelVolume`/EQ/child list). No
  ticket re-file; folded here.
- `#3-stealthduck-mix-adjuster-readback` `OPEN` reporter — Additional evidence
  (audio.authoring SoundClass-tree + `SM_StealthDuck` two-adjuster mix +
  short-falloff ambient build; outcome tool_bug for the separate looping no-op
  judge-filed `B-create-sound-cue-looping-noop`). Reproduces the SoundMix half
  exactly: story step 4 created `SM_StealthDuck` seeding two class adjusters
  (Ambient → 0.4, SFX → 1.0) and step 7 asked to "confirm the SoundMix carries
  the two class adjusters". `get_audio_info { SM_StealthDuck }` returned only
  `modifierCount` (the call log labels it "SM read … modifierCount" then
  "SoundMix only gives modifierCount"), so the requested per-adjuster confirm
  ("Ambient ducked to 0.4, SFX held at 1.0") could not be read through the
  documented verifier and the agent pivoted to `property.get` on the mix's
  `SoundClassEffects` array — the same `property.get` fallback as `#2`, not the
  `asset.dump` pivot of `#1`. Friction note (verbatim): "on the SoundMix only
  gives modifierCount, so I fell back to property.get to confirm
  AttenuationShapeExtents.X=400 and the per-class adjuster values." Confirms the
  SoundMix `modifierCount`-only readback is unchanged. No re-file; folded here.
- `#4-soundclasseffects-name-guess-miss` `OPEN` reporter — Additional evidence
  (REALISM-mode `Mix_Combat` combat-duck build: SC_Master + 4 children,
  `Mix_Combat` with SC_Music/SC_Ambient ducking adjusters; outcome tool_bug for
  the judge-filed fade-drop `B-add-mix-modifier-fade-params-dropped`). Reproduces
  the SoundMix half exactly — `get_audio_info { Mix_Combat }` returned only
  `modifierCount` (friction note verbatim: "get_audio_info on a SoundMix returns
  only modifierCount, not the per-adjuster volume/fade values, so I had to read
  the engine SoundMix.h to find the array UPROPERTY name and use property.get to
  verify the actual adjusters") — but adds a **new wrinkle on the `property.get`
  workaround itself**: the array UPROPERTY name is undocumented in the overlay, so
  the agent's first pivot guessed **`SoundClasses`** and ate a hard error —
  `property.get { Mix_Combat, SoundClasses }` → `[PROPERTY_NOT_FOUND] Failed to
  resolve property 'SoundClasses' on object … Mix_Combat: Property 'SoundClasses'
  not found` — before correcting to **`SoundClassEffects`** (which then
  succeeded). The agent had to source-dive engine `SoundMix.h` to find the right
  name. So the fallback the prior entries point at (`property.get` on the mix's
  adjuster array) carries its own discovery cost when the overlay never names the
  `SoundClassEffects` property — a misuse-then-correct (one `is_error` PROPERTY_NOT_FOUND
  then a working variant) layered on top of the documented-readback-can't-confirm
  gap. Strengthens the docs fix: when the overlay points at the `property.get` /
  `asset.dump` fallback for SoundMix adjusters, it should **name the
  `SoundClassEffects` array property** so an agent forced onto the workaround
  doesn't guess `SoundClasses` and re-discover the engine field name. No re-file;
  folded here.
- `#5-describe-verbs-shipped` `IN-REVIEW` developer — Reworded OPEN->IN-REVIEW.
  Rescoped from the interim `asset.dump`/`property.get` docs note (which the
  overlay's own `## Inspect-after-mutate` guidance now steers AWAY from) to the
  established describe-verb pattern the SoundAttenuation sibling shipped, then
  implemented it. Two new live readers in
  `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`
  (inserted after `describe_attenuation`, before `get_audio_info`):
  **`audio.authoring.describe_sound_class`** echoes the full
  `FSoundClassProperties` leaf surface — `volume`/`pitch`/`lowPassFilterFrequency`/
  `lfeBleed`/**`voiceCenterChannelVolume`** — plus `parentClass`, MCP_HAS_SUBMIX-gated
  `outputSubmix`, and a **`childClasses`** array (closes the readback half
  `B-sound-class-parent-no-child-link` deferred here); and
  **`audio.authoring.describe_sound_mix`** echoes `modifierCount` plus an
  **`adjusters`** array (`soundClass`/`volumeAdjuster`/`pitchAdjuster`/
  `applyToChildren` per `SoundClassEffects` entry) and the mix-level
  `applyEQ`/`eqPriority`/`fadeInTime`/`fadeOutTime`. Per-adjuster fade keys are
  deliberately NOT emitted — `FSoundClassAdjuster` has no fade member
  (`B-add-mix-modifier-fade-params-dropped` is the separate write-side bug).
  Overlay `docs/wiki-src/audio.authoring.md` updated: `## Inspect-after-mutate`
  now names both readers alongside `describe_attenuation` and states the
  `get_audio_info` SoundClass/SoundMix thinness, plus per-verb H3 sections.
  Regression tests added to
  `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Tests/Media/TestAudioHandlers.cpp`:
  `describe_sound_class.EchoesWrittenFields` drives production
  `set_class_properties` (writes `voiceCenterChannelVolume=0.75`) + `set_class_parent`,
  then asserts `describe_sound_class` echoes `voiceCenterChannelVolume`/`parentClass`
  on the child and the `childClasses` array on the parent;
  `describe_sound_mix.EchoesAdjusters` drives `add_mix_modifier`
  (`volumeAdjuster=0.4`/`pitchAdjuster=1.2`/`applyToChildren=false`) then asserts the
  `adjusters[0]` object round-trips all four fields; plus `MissingRequiredParam`
  early-return tests for both verbs. Each fails if the new handler is reverted
  (handler-not-found) or drops a field. Not yet compiled/tested (a later phase
  drives green).
- `#6-dry-mix-bool-flags-uncovered-both-sides` `IN-REVIEW` reporter — Additional
  evidence (REALISM-mode "low-health duck" SoundClass-hierarchy + SoundMix build;
  outcome ergo, folded here, no re-file). New angle the prior entries don't cover:
  the **boolean routing flags** of `FSoundClassProperties` —
  `bReverb` ("Send to Master Reverb Submix"), `bApplyAmbientVolumes`
  ("Interior/Exterior modifiers"), `bApplyEffects` ("Output to Master EQ Submix"),
  also `bIsMusic`/`bAlwaysPlay`/`bCenterChannelOnly` (engine `SoundClass.h:95-119`)
  — are exposed by **NEITHER** the typed write verb `set_class_properties` (its
  params are only `volume`/`pitch`/`lowPassFilterFrequency`/`lfeBleed`/
  `voiceCenterChannelVolume`/`parentSubmix` — wiki `set_class_properties.md:12-17`)
  **NOR** the just-shipped reader `describe_sound_class`. The story's step 3 ("keep
  Music dry: set its reverb/EQ/passive flags off") was only completable by
  source-diving engine `SoundClass.h` for the UPROPERTY names and pivoting to
  generic `property.set`/`property.get` on nested `Properties.bReverb` etc. — so the
  describe-verb fix shipped in `#5` does NOT close the dry-mix case the typical
  audio-mixing task needs. Two concrete confirmations via live replay against the
  same plugin:
  (a) **Reader omits the flags** — after `create_sound_class { ReplayMusic,
  /Game/Audio/MixReplay, vol 0.7 }` + `property.set Properties.bReverb=false` +
  `property.set Properties.bApplyEffects=false`,
  `describe_sound_class { /Game/Audio/MixReplay/ReplayMusic }` returns verbatim
  `{"type":"SoundClass","volume":0.699999988079071,"pitch":1,
  "lowPassFilterFrequency":20000,"lfeBleed":0,"voiceCenterChannelVolume":0,
  "childClasses":[],...}` — NO `bReverb`/`bApplyAmbientVolumes`/`bApplyEffects`
  (nor `bIsMusic`/`AttenuationDistanceScale`/`Default2DReverbSendAmount`), so the
  dry-mix tune the agent just wrote cannot be confirmed in-namespace. The generic
  `property.get { Properties.bReverb }` is the only readback (returns `value:false`,
  works).
  (b) **Reader's description over-promises** — the generated wiki Notes for
  `describe_sound_class` (live page `…/wiki/audio.authoring.describe_sound_class.md`,
  the handler-registered description string) claim it returns **"the full live
  `FSoundClassProperties` + hierarchy surface"** and "the fields
  `set_class_properties` writes" — but the payload is a strict subset (5 leaf floats
  + parent/child + submix), omitting the 7 boolean flags and 3 floats above. An
  agent trusting "full FSoundClassProperties surface" would wrongly conclude the
  dry-mix flags are unset/absent. Quotable mismatch: description says "full …
  FSoundClassProperties surface"; payload omits `bReverb`/`bApplyAmbientVolumes`/
  `bApplyEffects`. Strengthens the `#5` fix two ways: (1) widen
  `set_class_properties` AND `describe_sound_class` to cover the boolean routing
  flags (at least `bReverb`/`bApplyAmbientVolumes`/`bApplyEffects`/`bIsMusic`) so the
  common "keep a class dry" task stays typed; (2) until then, soften the
  `describe_sound_class` description from "the full live FSoundClassProperties
  surface" to the actual field list, and have the overlay name `property.set`/
  `property.get` on `Properties.<bFlag>` as the dry-mix workaround. No re-file.
- `#7-dry-mix-flags-recur-typed-setter-source-dive` `IN-REVIEW` reporter —
  Additional evidence (REALISM-mode "low-health mix profile" SoundClass-hierarchy
  + `LowHealthMix` SoundMix build; outcome ergo, folded here, no re-file).
  Reproduces the `#6` write-side gap exactly with a clean, error-free call log
  (every call `ok:true`, zero `is_error`/retries) — so the friction is purely
  process, not a tool failure. Story step 3 ("keep Music dry: set its
  passive/EQ-relevant flags off") was again only completable by **leaving the
  typed audio surface entirely**: the agent Read engine `SoundClass.h` to recover
  the UPROPERTY names (call-log entry verbatim "find dry-mix flag names
  bReverb/bApplyAmbientVolumes/bApplyEffects"), then issued 3 generic
  `property.set` (`Properties.bReverb=false`, `Properties.bApplyAmbientVolumes=false`,
  `Properties.bApplyEffects=false`) and, at verify time, 3 more generic
  `property.get` to confirm them — 6 generic property calls + one engine-source
  read standing in for what should be a single typed param on
  `set_class_properties` and a single field on `describe_sound_class`. Friction
  note (verbatim): "the dry-mix flags (bReverb/bApplyAmbientVolumes/bApplyEffects)
  are not exposed by the typed set_class_properties NOR surfaced by
  describe_sound_class, so I had to read SoundClass.h for the exact UPROPERTY
  names and use generic property.set/property.get on nested Properties.* paths to
  set and verify them - a discoverability/coverage gap for keeping a class dry,
  but no retries or errors." Confirms the `#6` two-sided gap is unchanged and that
  the "keep a class dry" intent — the single most common reason a mixing task
  touches a SoundClass beyond volume/pitch — still forces a source-dive + generic
  property pivot on BOTH the write and the verify side. Strengthens the `#6` ask
  to widen `set_class_properties` AND `describe_sound_class` to the boolean routing
  flags (at minimum `bReverb`/`bApplyAmbientVolumes`/`bApplyEffects`/`bIsMusic`);
  and, until then, to have `docs/wiki-src/audio.authoring.md` (the line at :83
  that names `set_class_properties` for "non-parent fields … etc." but never the
  dry-mix flags) name `property.set`/`property.get` on `Properties.<bFlag>` as the
  documented dry-mix path so the next agent skips the `SoundClass.h` dive. No
  re-file.
- `#8-describe-verbs-confirmed-live-clean` `IN-REVIEW` reporter — Positive
  confirmation of the `#5` describe-verb fix from live use (this is the FIRST
  evidence entry POST-ship; `#1`-`#4` all predate the fix and had to pivot to
  `asset.dump`/`property.get`). REALISM-mode `/Game/Audio` bus build
  (SM_Master+SM_SFX submixes, SC_Master+SC_SFX/SC_Music classes, SC_SFX tuned
  vol0.85/pitch1/lpf18000/output->SM_SFX, SC_Music vol0.7/centerVol1.0,
  Mix_Combat with SC_Music@0.4 + SC_SFX@1.1; outcome clean, every call `ok:true`,
  zero `is_error`/retries/hangs). The story step 6 asked to "Read everything back
  with get_audio_info" to confirm lpf==18000 and the per-adjuster 0.4/1.1 — the
  exact two fields this ticket documents `get_audio_info` cannot surface — and the
  agent, steered by the overlay, used the now-shipped `describe_sound_class` (for
  SC_SFX lpf, SC_Music centerVol) and `describe_sound_mix` (for Mix_Combat
  per-adjuster volumeAdjuster) to confirm them in-namespace, NO `asset.dump` /
  `property.get` pivot, NO source-dive. Friction note (verbatim): "get_audio_info
  by design omits lowPassFilterFrequency (classes) and the per-adjuster
  volumeAdjuster array (mixes) … I confirmed them via the wiki-documented
  describe_sound_class/describe_sound_mix readback methods (the wiki explicitly
  notes get_audio_info omits these and points to describe_*)." So both the
  handler readers AND the overlay's `## Inspect-after-mutate` routing
  (`audio.authoring.md:7`) land as intended — the verify path stayed in-namespace
  and clean. One residual MINOR process artifact: the agent emitted SIX readback
  calls (3 `get_audio_info` + 3 `describe_*`) where 3 `describe_*` would have
  sufficed — because the story literally named `get_audio_info` as the readback
  verb, the agent ran it first, got the thin payload, then supplemented with
  `describe_*`. The `describe_*` readers are a strict superset for SoundClass /
  SoundMix, so `get_audio_info` is now a redundant first hop whenever a leaf/
  adjuster value is wanted. Not worth its own ticket and the docs already steer
  correctly (overlay:7 says "use the `describe_*` readers to confirm a written
  value"); flagged only so the fix-side is aware the doubled-readback pattern
  persists while `get_audio_info` keeps its completeness-implying name. No re-file;
  this entry confirms the `#5` fix works from live use and folds the clean-run
  evidence here.
- `#9-combatduck-describe-confirms-modifiers` `IN-REVIEW` reporter — Additional
  post-ship positive confirmation (REALISM-mode "third-person action game"
  audio-mixing-foundation: SFX root + Weapons/Footsteps children, `CombatDuck`
  SoundMix seeding Footsteps@0.4 via `classAdjusters` + Weapons@1.2 via
  `add_mix_modifier`, `CombatHall` ReverbEffect; outcome tool_bug for the separate
  judge-filed disk-write defect `B-audio-create-save-no-disk-write`, and the
  ReverbEffect type-misreport filed separately as
  `E-get-audio-info-reverb-type-unknown #1` — same task). Reproduces the SoundMix
  half exactly AND confirms the `#5` describe-verb fix routes correctly: the
  story's final step said "verify the mix by reading the assets back with
  get_audio_info / describe calls", and the agent ran `get_audio_info { CombatDuck }`
  (call log: "read CombatDuck -> modifierCount 2" — the thin `modifierCount`-only
  payload this ticket documents), then pivoted **in-namespace** to
  `describe_sound_mix { CombatDuck }` (call log: "adjusters Footsteps0.4
  Weapons1.2") to confirm the per-adjuster values — NO `asset.dump` / `property.get`
  fallback, NO source-dive, every call `ok:true` with zero `is_error`/retries.
  Friction note (verbatim): "get_audio_info on a SoundMix returns only
  modifierCount (no per-adjuster values), so I needed describe_sound_mix to confirm
  the 0.4/1.2 adjusters". So both the handler reader and the overlay routing land
  as intended (second clean post-ship datapoint after `#8`). Reproduces the same
  residual `#8` flagged — `get_audio_info` is run first (because the story names it)
  then supplemented by `describe_sound_mix`, the doubled-readback the
  completeness-implying `get_audio_info` name keeps inviting. No re-file; folds the
  clean-run evidence here.
