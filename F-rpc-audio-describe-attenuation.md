---
id: F-rpc-audio-describe-attenuation
title: "Add live RPC `audio.authoring.describe_attenuation` — SoundAttenuation has 5 configure verbs but no live readback of what they wrote"
status: IN-REVIEW
severity: Medium
category: feature
tags: [audio, sound-attenuation, readback, dump-parity, inspect-after-mutate]
---

# Add live RPC `audio.authoring.describe_attenuation` for SoundAttenuation settings

`audio.authoring` ships a full SoundAttenuation authoring surface — `create_attenuation_settings` plus four `configure_*` mutators (`configure_distance_attenuation`, `configure_spatialization`, `configure_reverb_send`, `configure_occlusion`) — that write ~12 distinct `FSoundAttenuationSettings` fields onto the asset. But the only live readback, `audio.authoring.get_audio_info`, returns exactly **two** of them for a SoundAttenuation asset:

```cpp
// AudioAuthoringHandler.cpp:2567-2572 (get_audio_info, SoundAttenuation branch)
else if (USoundAttenuation* Atten = Cast<USoundAttenuation>(Asset))
{
    Result->SetStringField(TEXT("type"), TEXT("SoundAttenuation"));
    Result->SetNumberField(TEXT("falloffDistance"), Atten->Attenuation.FalloffDistance);
    Result->SetBoolField(TEXT("spatialize"), Atten->Attenuation.bSpatialize);
}
```

So `distanceAlgorithm`, `innerRadius` (written into `AttenuationShapeExtents.X`), `spatializationAlgorithm`, and the entire reverb block (`bEnableReverbSend`, `ReverbWetLevelMin/Max`, `ReverbDistanceMin/Max`) and occlusion block (`bEnableOcclusion`, `OcclusionLowPassFilterFrequency`, `OcclusionVolumeAttenuation`, `OcclusionInterpolationTime`) — every field the four `configure_*` verbs exist to set — have **no live read** that echoes them. An agent that runs the natural configure-then-confirm flow cannot verify its own writes through the namespace's own info verb.

SoundAttenuation is the conspicuous missing member of the audio describe family. SoundCue got `describe_sound_cue` (`F-rpc-audio-describe-sound-cue`, DONE), MetaSound got `describe_metasound` (`F-rpc-audio-describe-metasound`, DONE), and SoundWave got `describe_sound_wave` — each explicitly because `get_audio_info` is "the lightweight summary surface" and a structural live read was needed for inspect-after-mutate. The `audio.authoring` wiki overlay even points callers to those `describe_*` reads. But there is **no** `describe_attenuation` and **no** dedicated attenuation dump-sidecar builder: grepping `Source/` for `describe_attenuation`, `get_attenuation`, `attenuation_info`, `AttenuationDumpBuilder`, and `attenuation.json` returns nothing. The configured settings only surface via `asset.dump`'s generic `properties.json` (the full `FSoundAttenuationSettings` struct under the `Attenuation` UPROPERTY), which writes files to the on-disk dump cache.

This also brushes the asset-dump-parity policy the SoundCue/MetaSound describe RPCs were filed under (anything reachable in a dump should be reachable via a live RPC). Today the only way to confirm a configured attenuation profile live is to recognize `get_audio_info` is too thin and pivot to `asset.dump` → read `properties.json` off disk — exactly the workaround this ticket's repro task had to use.

**Verbatim repro (this task):** build `/Game/Audio/Attenuation/ForestAmbience_Attenuation` then confirm.

```
call("audio.authoring.create_attenuation_settings", { name:"ForestAmbience_Attenuation", path:"/Game/Audio/Attenuation", innerRadius:400, falloffDistance:6000 })
call("audio.authoring.configure_distance_attenuation", { assetPath:".../ForestAmbience_Attenuation", innerRadius:400, falloffDistance:6000, distanceAlgorithm:"naturalsound" })
call("audio.authoring.configure_spatialization",       { assetPath:".../ForestAmbience_Attenuation", spatialize:true, spatializationAlgorithm:"hrtf" })
call("audio.authoring.configure_reverb_send",          { assetPath:".../ForestAmbience_Attenuation", enableReverbSend:true, reverbWetLevelMin:0, reverbWetLevelMax:0.9, reverbDistanceMin:500, reverbDistanceMax:5000 })
call("audio.authoring.configure_occlusion",            { assetPath:".../ForestAmbience_Attenuation", enableOcclusion:true, occlusionLowPassFilterFrequency:800, occlusionVolumeAttenuation:0.4, occlusionInterpolationTime:0.2 })
call("audio.authoring.get_audio_info",                 { assetPath:".../ForestAmbience_Attenuation" })
// -> {"type":"SoundAttenuation","falloffDistance":6000,"spatialize":true,"message":"Audio info retrieved"}
```

All five configure calls succeeded and every value persisted (confirmed via `asset.dump` → `properties.json`: `DistanceAlgorithm=NaturalSound`, `AttenuationShapeExtents.X=400` (the field `innerRadius` writes), `bSpatialize=true`, `SpatializationAlgorithm=SPATIALIZATION_HRTF`, `bEnableReverbSend=true`, `ReverbWetLevelMin=0`/`Max≈0.9`, `ReverbDistanceMin=500`/`Max=5000`, `bEnableOcclusion=true`, `OcclusionLowPassFilterFrequency=800`, `OcclusionVolumeAttenuation≈0.4`, `OcclusionInterpolationTime≈0.2`). The writes are correct; the gap is purely the missing live readback.

**Fix:** Add `audio.authoring.describe_attenuation` (param: `assetPath`) next to `get_audio_info` in `AudioAuthoringHandler.cpp`. Load via the existing `LoadSoundAttenuationFromPath()` helper (already in the file, line 268), then serialize the `FSoundAttenuationSettings` struct (e.g. via `FJsonObjectConverter::UStructToJsonObject` against `Atten->Attenuation`, or hand-roll the fields the four `configure_*` verbs write — note `innerRadius` lives in `AttenuationShapeExtents.X`, not a `RadiusMin` field) and return it. Error `ATTENUATION_NOT_FOUND` on load failure, mirroring the configure verbs. Keep `get_audio_info` as the lightweight summary surface; `describe_attenuation` is the full readback, parallel to the SoundCue / MetaSound / SoundWave describe RPCs. Update the `audio.authoring` wiki "Inspect-after-mutate" line to list it.

**Workaround:** confirm configured attenuation settings via `asset.dump { assetPath }` and read `properties.json` (the full `FSoundAttenuationSettings` lives under the `Attenuation` UPROPERTY), not `get_audio_info`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed the full ForestAmbience_Attenuation configure flow (all 5 verbs `ok:true`, all values persisted per `asset.dump` properties.json). `audio.authoring.get_audio_info` on a SoundAttenuation returns only `falloffDistance`+`spatialize` (AudioAuthoringHandler.cpp:2567-2572), echoing none of the distance-algorithm / spatialization-algorithm / reverb / occlusion fields the four `configure_*` verbs write. No `describe_attenuation` live RPC and no attenuation dump-sidecar builder exist (grep of `Source/` for describe_attenuation / get_attenuation / AttenuationDumpBuilder / attenuation.json — empty), so the only live confirm is `asset.dump` → `properties.json`. Same parity-family gap as `F-rpc-audio-describe-sound-cue` (DONE) and `F-rpc-audio-describe-metasound` (DONE); SoundAttenuation is the missing member. Proposed thin wrapper: `audio.authoring.describe_attenuation { assetPath }` serializing `FSoundAttenuationSettings`. Seed method was `audio.authoring.configure_reverb_send`, which works correctly; the finding is about the missing readback (culprit: get_audio_info).
- `#2-reword` `OPEN` developer — Corrected stale cites before implementing: get_audio_info SoundAttenuation branch is at AudioAuthoringHandler.cpp:2567-2572 (not 2445-2450), LoadSoundAttenuationFromPath is at :268 (not :218), and the `innerRadius` value is written into `AttenuationShapeExtents.X` — there is no `RadiusMin` field on this path (configure_distance_attenuation.cpp:1787). Updated the snippet header, repro confirmation, and Fix accordingly.
- `#3-implemented` `IN-REVIEW` developer — Added live RPC `audio.authoring.describe_attenuation { assetPath }` next to `get_audio_info` in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`. Loads via the existing `LoadSoundAttenuationFromPath()` and hand-rolls the full configure surface the four `configure_*` verbs write: `distanceAlgorithm` (lowercase token), `falloffDistance`, `innerRadius` (= `AttenuationShapeExtents.X`), `spatialize`, `spatializationAlgorithm` (token), the occlusion block (`enableOcclusion`, `occlusionLowPassFilterFrequency`, `occlusionVolumeAttenuation`, `occlusionInterpolationTime`), and the reverb-send block (`enableReverbSend`, `reverbWetLevelMin/Max`, `reverbDistanceMin/Max`). Returns `ATTENUATION_NOT_FOUND` on load failure, mirroring the configure verbs; `get_audio_info` is left unchanged as the lightweight summary. Regression test `EditorAutomationRpcGateway.audio.authoring.describe_attenuation.EchoesConfiguredFields` (plus a `.MissingRequiredParam` early-return test) added in `Private/Tests/Media/TestAudioHandlers.cpp`: it drives the four production `configure_*` verbs through the dispatcher, then asserts `describe_attenuation` echoes every field — it fails (handler-not-found) if the new RPC is reverted. Updated the `audio.authoring` wiki overlay `## Inspect-after-mutate` line and added a `### audio.authoring.describe_attenuation` H3. No aspect-version bump (brand-new live RPC, no dump-cache aspect changed).
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
