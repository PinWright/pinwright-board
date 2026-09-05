---
id: B-sound-wave-properties-stale-derived-state
title: "audio.authoring.set_sound_wave_properties changes USoundWave fields without rebuilding compressed data or publishing the updated runtime proxy"
status: IN-REVIEW
severity: High
category: bug
tags: [audio, sound-wave, compression, derived-state, runtime, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# Sound-wave property edits leave engine-derived audio state stale

## What happens

`audio.authoring.set_sound_wave_properties` writes `bLooping`, volume, pitch,
`SoundGroup`, `CompressionQuality`, and flags directly; the private compression
field is written through `FIntProperty::SetPropertyValue_InContainer`
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\SoundWaveAuthoringHandler.cpp:67-110`).
The handler then only marks the asset dirty and reports direct field getters at
`:112-133`. It never sends `PostEditChangeProperty` or invokes an equivalent
sound-wave refresh.

The engine's normal `USoundWave::PostEditChangeProperty` path treats
`CompressionQuality` specially by calling `UpdateAsset`, and refreshes other
changes with `FSoundWaveData::InitializeDataFromSoundWave`
(`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\SoundWave.cpp:3267-3272,3300-3407`).
That derived object caches loop state and the compressed-data GUID at `:253-284`,
while `UpdatePlatformData` publishes the replacement data to the runtime proxy at
`:4280-4303`.

## Why it matters

The response can echo the new compression quality or loop flag while compressed
audio and the live proxy still represent the previous settings. Callers may save
and audition a stale result, mistaking a direct UPROPERTY readback for effective
runtime state. This is High because a normal setter silently skips the engine's
required derived-state path.

## What should happen

For every changed field, issue the correct `FPropertyChangedEvent` through
`PostEditChangeProperty`, or use a shared `USoundWave` setter/refresh helper that
preserves the same `UpdateAsset` and proxy-publication behavior. Perform this
before save and response, and report an effective post-refresh readback. This is
the catalog's `mutation-skips-derived-bookkeeping` / `derived-state-not-refreshed`
fix shape, not another direct-field assignment.

## Workaround

Change the affected property once in the SoundWave editor to trigger the native
post-edit path, then save and re-audition the asset.

## Fix

Changed `SoundWaveAuthoringHandler.cpp` to notify an actual changed
SoundWave property through `PinWright::NotifyPropertyChanged` before saving or
responding. Compression-quality edits use the engine's `UpdateAsset` branch;
ordinary property batches publish the refreshed metadata with
`UpdatePlatformData()`, while `save:false` restores a previously clean package
after the engine refresh. Added the tick-unsafe dispatcher registration and the
handler-harness regressions
`PinWright.audio.authoring.set_sound_wave_properties.PublishesOrdinaryProperties`
and
`PinWright.audio.authoring.set_sound_wave_properties.RefreshesCompressionQuality`,
which independently check runtime proxy publication and compressed-data GUID
refresh.

## Related

- Data-loss pattern `mutation-skips-derived-bookkeeping`.
- False-success patterns `derived-state-not-refreshed` and `request-echo-not-result-readback`.
- `F-sound-wave-property-edit` — introduced and field-echo tested this setter but does not cover derived audio state.
- `E-describe-sound-wave-omits-compression` — readback sibling, not runtime refresh.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan followed the handler's direct property writes and field echo into the UE 5.8 `USoundWave::PostEditChangeProperty` implementation, which performs the omitted `UpdateAsset`, `InitializeDataFromSoundWave`, and proxy publication work. Board search found only field-authoring/readback tickets, not this derived-state mechanism. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
- `#2-refresh-derived-state` `IN-REVIEW` developer — Changed `SoundWaveAuthoringHandler.cpp` to run the public SoundWave property-notification path and publish ordinary-property refreshes; added `PinWright.audio.authoring.set_sound_wave_properties.RefreshesDerivedState` to assert derived loop state and the compressed-data GUID. Static verification only; no build, test, editor, or MCP run was performed.
- `#3-preserve-save-state` `IN-REVIEW` developer — Preserved a previously clean package when `save:false` while retaining the engine refresh, and extended the handler-harness regression to assert that contract. Static verification only; no build, test, editor, or MCP run was performed.
- `#4-safepoint-independent-tests` `IN-REVIEW` developer — Registered `audio.authoring.set_sound_wave_properties` as tick-unsafe because compression refresh reaches synchronous `UpdateAsset`; split the behavioral coverage into independent ordinary-publication and compression-quality tests, and replaced raw root ownership with `TStrongObjectPtr`. Static verification only; no build, test, editor, or MCP run was performed.
