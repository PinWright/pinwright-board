---
id: B-add-mix-modifier-fade-params-dropped
title: "audio.authoring.add_mix_modifier registers fadeInTime/fadeOutTime params, accepts them, returns success — but the handler never writes them onto the USoundMix (USoundMix::FadeInTime/FadeOutTime), so both values are silently dropped from the saved asset"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, soundmix, add_mix_modifier, silent-drop, schema, success-no-effect, fade]
---

# `audio.authoring.add_mix_modifier` silently drops `fadeInTime` / `fadeOutTime`

`audio.authoring.add_mix_modifier` advertises `fadeInTime` and `fadeOutTime` as
first-class optional number parameters (in both its RPC schema and the generated
wiki page), accepts them with no error, and returns `{"message":"Mix modifier
added", ... "existsAfter":true}` — but the handler **never reads either field**.
The result is a clean success that quietly discards two documented inputs.

The fade times are **mix-level**, not per-adjuster: `FSoundClassAdjuster` (the
per-class struct this handler appends to `SoundClassEffects[]`) has no fade
members, but the enclosing **`USoundMix` asset DOES persist them** as
`float FadeInTime` (engine `Sound/SoundMix.h:196`) and `float FadeOutTime`
(`SoundMix.h:204`) — both `EditAnywhere, Category=SoundMix` UPROPERTYs saved on
the asset, alongside `InitialDelay` (:192) and `Duration` (:200). The handler
already loads the `USoundMix*` (`AudioAuthoringHandler.cpp:1562`); it simply
never writes `Mix->FadeInTime`/`Mix->FadeOutTime` before saving. So the values
the caller passed are perfectly persistable — they are just dropped on the floor.

This is the same **documented-input-silently-dropped** shape as
`B-add-montage-notify-time-dropped` and `B-add-variable-default-value-ignored`,
here on a SoundMix modifier's fade times. It is distinct from the
readback-thinness ergonomic ticket `E-audio-get-info-soundclass-mix-readback-thin`
(which is about `get_audio_info` not echoing the adjuster values it *does*
persist) — this ticket is that the param is accepted-and-discarded at the
**write** site, not merely unreadable.

## Source confirmation

`Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`:

- The handler **registers** the two params (lines 1552-1553):
  ```cpp
  RPC_PARAM_DEF("fadeInTime", "number", "Fade in time", "0"),
  RPC_PARAM_DEF("fadeOutTime", "number", "Fade out time", "0"),
  ```
- But the body only assigns volume / pitch / applyToChildren onto the adjuster
  and never references `fadeInTime` / `fadeOutTime` again (lines 1576-1582):
  ```cpp
  FSoundClassAdjuster Adjuster;
  Adjuster.SoundClassObject = SoundClass;
  Adjuster.VolumeAdjuster   = static_cast<float>(Ctx.GetNumber(TEXT("volumeAdjuster"), 1.0));
  Adjuster.PitchAdjuster    = static_cast<float>(Ctx.GetNumber(TEXT("pitchAdjuster"), 1.0));
  Adjuster.bApplyToChildren = Ctx.GetBool(TEXT("applyToChildren"), true);
  Mix->SoundClassEffects.Add(Adjuster);
  ```
  `FSoundClassAdjuster` (engine `Sound/SoundMix.h:131`, NOT `SoundClass.h`) has no
  `FadeInTime`/`FadeOutTime` member — correct. But the **mix asset itself** does:
  `USoundMix::FadeInTime` (`SoundMix.h:196`) and `USoundMix::FadeOutTime`
  (`SoundMix.h:204`) are persisted UPROPERTYs. The handler holds `Mix` (loaded at
  `:1562`) and could write them directly; it just doesn't. `create_sound_mix`
  (`:1478-1535`) likewise never sets them, so today there is no path to author a
  mix fade at all.

## Verbatim live repro (replayed against mcp__editor-automation__call)

1. `audio.authoring.create_sound_class { name: SC_FadeProbe, path: /Game/AudioFadeProbe/Classes }` → ok
2. `audio.authoring.create_sound_mix { name: Mix_FadeProbe, path: /Game/AudioFadeProbe/Mixes }` → ok
3. `audio.authoring.add_mix_modifier { assetPath: /Game/AudioFadeProbe/Mixes/Mix_FadeProbe, soundClassPath: /Game/AudioFadeProbe/Classes/SC_FadeProbe, volumeAdjuster: 0.35, fadeInTime: 1.5, fadeOutTime: 2, applyToChildren: true }`
   → `{"message":"Mix modifier added","assetPath":"/Game/AudioFadeProbe/Mixes/Mix_FadeProbe","existsAfter":true,"assetClass":"SoundMix"}` — success, no warning that the two fade params are inert.
4. `property.get { objectPath: /Game/AudioFadeProbe/Mixes/Mix_FadeProbe.Mix_FadeProbe, propertyName: SoundClassEffects }`
   → `{"value":[{"SoundClassObject":"/Game/AudioFadeProbe/Classes/SC_FadeProbe.SC_FadeProbe","VolumeAdjuster":0.3499999940395355,"PitchAdjuster":1,"LowPassFilterFrequency":20000,"bApplyToChildren":true,"VoiceCenterChannelVolumeAdjuster":1}]}`

The persisted adjuster has the volume (0.35) and `bApplyToChildren` that were
accepted, but **no fadeInTime (1.5) or fadeOutTime (2) anywhere** — the two
documented inputs were swallowed silently. An agent that follows the schema to
author a combat-duck fade has no signal that the fade never persisted.

## What it should do

A documented, accepted parameter must not be silently discarded. Because the
fade IS asset-persistable on the `USoundMix`, the fix is to **make the write
work**, the same resolution the sibling silent-drop tickets used
(`B-add-montage-notify-time-dropped`, `B-add-variable-default-value-ignored`):

- **Fix:** in `add_mix_modifier`, write the mix-level fade onto the loaded
  `USoundMix` before `SaveAudioAsset(Mix, bSave)`:
  ```cpp
  Mix->FadeInTime  = static_cast<float>(Ctx.GetNumber(TEXT("fadeInTime"),  Mix->FadeInTime));
  Mix->FadeOutTime = static_cast<float>(Ctx.GetNumber(TEXT("fadeOutTime"), Mix->FadeOutTime));
  ```
  Default to the mix's current value so repeated `add_mix_modifier` calls that
  omit the fades don't clobber a previously-set fade. Echo the persisted
  `fadeInTime`/`fadeOutTime` back in the result so the success payload is
  truthful.
- Since the fade is a property of the mix, not the modifier, also expose the same
  two optional params on `create_sound_mix` so a fade can be authored at creation
  time (it currently sets none of `FadeInTime`/`FadeOutTime`/`InitialDelay`/
  `Duration`).

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode audio-mixing-bus build
  (SoundClass hierarchy + `Mix_Combat` combat-duck SoundMix with per-class
  fade-in adjusters). `add_mix_modifier` registers `fadeInTime`/`fadeOutTime`
  (`AudioAuthoringHandler.cpp:1552-1553`) but the body (`:1576-1582`) never reads
  them and `FSoundClassAdjuster` has no fade field, so they are silently dropped.
  Replay-confirmed live: `add_mix_modifier {... fadeInTime:1.5, fadeOutTime:2}`
  returned `"Mix modifier added"` with no warning, then
  `property.get { SoundClassEffects }` showed the persisted adjuster carrying
  `VolumeAdjuster 0.35` and `bApplyToChildren true` but **no fade values at all**
  — a documented input accepted and discarded with a success result. Distinct
  from `E-audio-get-info-soundclass-mix-readback-thin` (a readback-thinness ergo
  on `get_audio_info`); this is a write-side silent-drop on a documented param,
  same shape as `B-add-montage-notify-time-dropped` /
  `B-add-variable-default-value-ignored`.
- `#2-reword+fix` `IN-REVIEW` developer — REWORDED then fixed. The original ticket's
  premise ("fade-in/out are runtime mix-activation params on `PushSoundMixModifier`,
  not asset-persistable — drop/reject the params") was factually wrong: `USoundMix`
  itself persists `float FadeInTime` (engine `Sound/SoundMix.h:196`) and
  `float FadeOutTime` (`:204`) as `EditAnywhere, Category=SoundMix` UPROPERTYs. The
  fade is mix-level, not per-`FSoundClassAdjuster`. Reworded title/body/**Fix:** to
  "persist onto `USoundMix::FadeInTime`/`FadeOutTime`" — the make-it-work resolution
  both sibling silent-drop tickets used. Fix in
  `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`:
  `add_mix_modifier` (`:1576-1591`) now writes `Mix->FadeInTime`/`Mix->FadeOutTime`
  from the params (defaulting to the mix's current value so an omitted fade doesn't
  clobber a previously-set one) before `SaveAudioAsset` and echoes both back in the
  result; `create_sound_mix` (`:1478-1545`) gained matching optional
  `fadeInTime`/`fadeOutTime` params writing the same two UPROPERTYs so a mix fade is
  authorable at creation. Regression test
  `EditorAutomationRpcGateway.audio.authoring.add_mix_modifier.PersistsFade` in
  `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestAudioHandlers.cpp`
  creates a mix + class (save=false), asserts both fade fields start at 0, calls
  `add_mix_modifier { fadeInTime:1.5, fadeOutTime:2 }`, then asserts
  `Mix->FadeInTime==1.5`/`FadeOutTime==2`, the result echoes both, and a second
  fade-less `add_mix_modifier` leaves them unchanged — every assertion regresses to
  false if the writes are reverted.
