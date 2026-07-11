---
id: B-cue-setter-missing-asset-silent-success
title: "audio.authoring.set_cue_concurrency / set_cue_attenuation report success when the target asset path fails to load — silent no-op leaves the cue unchanged"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, sound-cue, set_cue_concurrency, set_cue_attenuation, silent-noop, false-success, validation]
encounters: 1
lastSeen: 2026-07-11T02:58:50.8889687+03:00
---

# `audio.authoring.set_cue_concurrency` / `set_cue_attenuation` fake-succeed when their asset path fails to load

Both SoundCue single-reference setters take an optional asset path
(`concurrencyPath` / `attenuationPath`) and treat "empty" as "clear". They load
the referenced asset with `StaticLoadObject`, but the assignment is gated behind
an unchecked `if (Loaded)` with **no `else`**: when the path is non-empty but
resolves to `nullptr` (typo, wrong path, or an asset that does not exist yet),
the handler assigns nothing, then falls straight through to `SaveAudioAsset` and
`SendSuccess` with `"Concurrency settings updated"` / `"Attenuation settings
updated"`. There is no `CONCURRENCY_NOT_FOUND` / `ATTENUATION_NOT_FOUND` error —
even though the SAME handlers raise `CUE_NOT_FOUND` when the cue itself is
missing. The caller gets a green "updated" and reasonably concludes the cue is
now governed by that concurrency group / attenuation, when in fact:

- if the cue previously had NO such setting, it still has none (the requested
  throttle silently does not exist), or
- if the cue previously HAD one, it silently keeps the OLD value (a stale-wrong
  lie — caller believes it points at the new group).

This is exactly the failure the manipulation-SFX designer hit: pointing a cue at
a shared concurrency group before that group existed returned "updated", so the
clipping the group was meant to cap silently keeps happening in-game.

Affected methods (shared root cause — same skip-on-load-failure shape):

- `audio.authoring.set_cue_concurrency` (`concurrencyPath`)
- `audio.authoring.set_cue_attenuation` (`attenuationPath`)

## What it should do

When the path parameter is present but non-empty and the asset fails to load,
reject with a domain error before saving — e.g.
`SendError("CONCURRENCY_NOT_FOUND", "Could not load SoundConcurrency: <path>")`
and the `ATTENUATION_NOT_FOUND` analogue — mirroring the existing `CUE_NOT_FOUND`
guard in the same handlers. The empty-string "clear" path must still succeed.

## Verbatim repro (live, replayed against mcp__pinwright__call)

Concurrency (cue `SC_Manip_Scale`, its ConcurrencySet was empty going in):

1. `audio.authoring.set_cue_concurrency { "assetPath": "/Game/Audio/Cues/SC_Manip_Scale", "concurrencyPath": "/Game/Audio/Concurrency/CG_DoesNotExist_ReplayProbe", "save": true }`
   -> `{"message":"Concurrency settings updated","assetPath":"/Game/Audio/Cues/SC_Manip_Scale","assetName":"SC_Manip_Scale","existsAfter":true,"assetClass":"SoundCue"}`  (green success)
2. `audio.authoring.decompile_sound_cue { "assetPath": "/Game/Audio/Cues/SC_Manip_Scale" }`
   -> SCIR body has NO `concurrency` line — the ConcurrencySet is still empty; nothing was applied.

Attenuation (same cue):

3. `audio.authoring.set_cue_attenuation { "assetPath": "/Game/Audio/Cues/SC_Manip_Scale", "attenuationPath": "/Game/Audio/Attenuation/ATT_DoesNotExist_ReplayProbe", "save": true }`
   -> `{"message":"Attenuation settings updated", ... "assetClass":"SoundCue"}`  (green success)
4. `audio.authoring.get_audio_info { "assetPath": "/Game/Audio/Cues/SC_Manip_Scale" }`
   -> `{"assetPath":".../SC_Manip_Scale","assetClass":"SoundCue","type":"SoundCue","duration":2.7799792,"nodeCount":1,"message":"Audio info retrieved"}`  — no `attenuationPath` field, so `AttenuationSettings` is still null; nothing was applied.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`

`set_cue_concurrency` (L775-788) — non-empty path loads to `nullptr`, `if (Conc)` skips, no `else`, then unconditional save + success:

```cpp
775    if (!ConcurrencyPath.IsEmpty())
776    {
777        USoundConcurrency* Conc = Cast<USoundConcurrency>(
778            StaticLoadObject(USoundConcurrency::StaticClass(), nullptr, *NormalizeAudioPath(ConcurrencyPath)));
779        if (Conc)
780        {
781            Cue->ConcurrencySet.Empty();
782            Cue->ConcurrencySet.Add(Conc);
783        }
784    }
785    else
786    {
787        Cue->ConcurrencySet.Empty();
788    }
```

then L790-795: `SaveAudioAsset(Cue, bSave);` ... `Result->SetStringField(TEXT("message"), TEXT("Concurrency settings updated"));` ... `Ctx.SendSuccess(Result);`.

`set_cue_attenuation` (L737-745) — identical shape:

```cpp
737    if (!AttenuationPath.IsEmpty())
738    {
739        USoundAttenuation* Atten = LoadSoundAttenuationFromPath(AttenuationPath);
740        if (Atten) Cue->AttenuationSettings = Atten;
741    }
742    else
743    {
744        Cue->AttenuationSettings = nullptr;
745    }
```

then L747-752 saves and returns `"Attenuation settings updated"`. `LoadSoundAttenuationFromPath` (L297-301) returns `Cast<USoundAttenuation>(StaticLoadObject(...))` — `nullptr` on a missing asset. Contrast the same handlers' `CUE_NOT_FOUND` guard (L769-773 / L731-735) which DOES fail loud when the cue is missing.

severity rationale: impact=silent false-success / stale-wrong on a normal path (High) x reach=rare (missing-asset edge on a non-every-session cue setter) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode replay of `audio.authoring.set_cue_concurrency` (shared-manipulation-SFX concurrency task). Confirmed live against `mcp__pinwright__call`: a non-existent `concurrencyPath` returns `{"message":"Concurrency settings updated"}` while `decompile_sound_cue` shows the ConcurrencySet unchanged (still empty); the sibling `set_cue_attenuation` behaves identically (`"Attenuation settings updated"` on a non-existent `attenuationPath`, `get_audio_info` shows no `attenuationPath`). Family root cause read in `AudioAuthoringHandler.cpp`: both setters gate the assignment behind `if (Loaded)` with no `else`, so a failed `StaticLoadObject` skips silently and the handler still saves + reports success — no `CONCURRENCY_NOT_FOUND` / `ATTENUATION_NOT_FOUND` despite the same handlers raising `CUE_NOT_FOUND` for a missing cue. Same misleading-success class as `B-set-aa-invalid-method-silent-noop`. Proposed: fail loud (domain error) when a non-empty path fails to load, keeping the empty-string "clear" path a success.
- `#2-fix-fail-loud` `IN-REVIEW` developer — Added the missing `else`-guard to both SoundCue single-reference setters in `Plugins/PinWright/Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`. `set_cue_attenuation` now rejects a non-empty `attenuationPath` that fails to load with `ATTENUATION_NOT_FOUND` (mirroring the `configure_*` attenuation setters), and `set_cue_concurrency` rejects a failed `concurrencyPath` load with `CONCURRENCY_NOT_FOUND` (matching the in-file `*_NOT_FOUND` convention). Both error BEFORE `SaveAudioAsset` so nothing is written, both preserve the empty-string "clear" path as a success, and both mirror the sibling `CUE_NOT_FOUND` guard already in the same handlers. Regression tests added in `Plugins/PinWright/Source/PinWright/Private/Tests/Media/TestAudioHandlers.cpp`: `PinWright.audio.authoring.set_cue_attenuation.RejectsMissingAsset` and `PinWright.audio.authoring.set_cue_concurrency.RejectsMissingAsset` — each drives the production handler through the dispatcher against a live transient SoundCue and asserts (A) missing sub-asset → domain error + nothing applied, (B) valid sub-asset → applied + success, (C) empty path → clear succeeds. Assertion (A) regresses to a fake success if the guard is reverted.
