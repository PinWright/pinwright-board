---
id: F-source-effect-preset-authoring
title: "No RPC to create a USoundEffectSourcePreset — add_source_effect dead-ends without one"
status: OPEN
severity: Medium
category: feature
tags: [audio, source-effect, authoring, missing-preset-creator]
encounters: 1
lastSeen: 2026-07-02T06:38:03.8599329+03:00
---

# No RPC to create a USoundEffectSourcePreset — add_source_effect dead-ends without one

The `audio.authoring` namespace can create a source-effect chain
(`create_source_effect_chain` → a `USoundEffectSourcePresetChain`) and add
entries to it (`add_source_effect`), but it ships **no verb to create the
`USoundEffectSourcePreset` assets those entries require**. As a result the
two-verb source-effect authoring surface is a dead end: you can make the empty
chain but cannot populate it via the RPC surface at all.

`add_source_effect` only appends an entry when `effectPresetPath` resolves to an
already-existing `USoundEffectSourcePreset`. Its `effectType` parameter is inert
(documented "(informational)" and never read on the live path — see source
below), so there is no "add a HighPassFilter/EQ/BitCrusher effect by type" shortcut
that would fabricate the preset. With no preset asset on disk and no RPC to make
one, every call fails `[PRESET_NOT_FOUND]`. The only way to obtain the prerequisite
preset assets is to drop out of the RPC surface entirely into
`python.execute` (`NewObject`-ing `USourceEffectFilterPreset` /
`USourceEffectEQPreset` / `USourceEffectBitCrusherPreset` etc. by hand), which is a
raw-scripting escape hatch, not a supported authoring capability.

This mirrors the already-fixed `F-audio-submix-asset-authoring` on the *submix*
side (that ticket added `create_sound_submix` and wired routing) — the
**source-effect** side has the same "one verb short of a working feature" hole,
just for `USoundEffectSourcePreset` instead of submix assets.

## Verbatim repro (replayed at HEAD via mcp__pinwright__call)

1. `audio.authoring.create_source_effect_chain { name: "SFXChain_OracleReplay", path: "/Game/Audio/Effects" }`
   → OK, `assetClass: "SoundEffectSourcePresetChain"`, chain exists.
2. `audio.authoring.add_source_effect { assetPath: "/Game/Audio/Effects/SFXChain_OracleReplay", effectType: "HighPassFilter", bypass: false }`
   → `[PRESET_NOT_FOUND] Effect preset path required or preset not found`
   (passing `effectType` alone does nothing — the param is inert).
3. `audio.authoring.add_source_effect { assetPath: "/Game/Audio/Effects/SFXChain_OracleReplay", effectPresetPath: "/Game/Audio/Effects/SFXP_HighPass_DoesNotExist", bypass: false }`
   → `[PRESET_NOT_FOUND] Effect preset path required or preset not found`

There is no `audio.authoring.create_source_effect_preset` (or any
`USoundEffectSourcePreset` creator) in the registry, and no generic
`asset.create` — grep of the whole handler tree for a source-effect-preset
registration returns only the chain creator.

## Guilty source (Plugins/PinWright/Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp)

`add_source_effect` — the preset must pre-exist; `effectType` is never read on the live path:

```cpp
2437:        RPC_PARAM_OPT("effectType", "string", "Type of effect (informational)"),
...
2461:    USoundEffectSourcePreset* EffectPreset = nullptr;
2462:    if (!EffectPresetPath.IsEmpty())
2463:    {
2464:        EffectPreset = Cast<USoundEffectSourcePreset>(
2465:            StaticLoadObject(USoundEffectSourcePreset::StaticClass(), nullptr, *NormalizeAudioPath(EffectPresetPath)));
2466:    }
...
2483:    else
2484:    {
2485:        Ctx.SendError(TEXT("PRESET_NOT_FOUND"), TEXT("Effect preset path required or preset not found"));
2486:    }
```

The only registered source-effect creator is the chain, not the preset:

```cpp
2381:REGISTER_RPC_HANDLER("audio.authoring.create_source_effect_chain", "audio.authoring", "Create a source effect preset chain",
```

severity rationale: impact=hard-blocker-with-workaround (python.execute preset
fabrication) × reach=rare (source-effect DSP is a specialized audio path, not
every-session) -> Medium.

**Workaround:** Fabricate the `USoundEffectSourcePreset` subclass assets
(`USourceEffectFilterPreset`, `USourceEffectEQPreset`,
`USourceEffectBitCrusherPreset`, …) via `python.execute` (`NewObject` +
`save_asset`), then pass each `effectPresetPath` to `add_source_effect`.

**Fix:** Add `audio.authoring.create_source_effect_preset(name, effectClass,
path?, save?)` that resolves a `USoundEffectSourcePreset` subclass (by short
name or `/Script/...` path — Filter/EQ/BitCrusher/etc.), `NewObject`s it into
`/Game/Audio/Effects` (mirroring `create_source_effect_chain`), and returns
asset verification. Then `create_source_effect_chain` → `create_source_effect_preset`
(×N) → `add_source_effect` closes the loop entirely inside the RPC surface.
Optionally, either make `add_source_effect`'s `effectType` create-and-append the
preset inline, or drop the inert param so it stops implying a by-type shortcut
that does not exist.

## History
- `#1-initial-repro` `OPEN` reporter — Filed against HEAD. `add_source_effect` requires a pre-existing `USoundEffectSourcePreset` (`AudioAuthoringHandler.cpp:2461-2486`) but no RPC creates one (only `create_source_effect_chain` at line 2381; no generic `asset.create`). `effectType` is inert (`:2437`, never read on the live `MCP_HAS_SOURCE_EFFECT` path). Replayed both dead-ends live: `effectType`-only and nonexistent-preset-path both return `[PRESET_NOT_FOUND] Effect preset path required or preset not found`. Task was only completable by fabricating the three preset assets via `python.execute`. Same class of gap as the now-DONE `F-audio-submix-asset-authoring`, on the source-effect side.
