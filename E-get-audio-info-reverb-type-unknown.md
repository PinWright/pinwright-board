---
id: E-get-audio-info-reverb-type-unknown
title: "audio.authoring.get_audio_info returns type:'Unknown' for a UReverbEffect (assetClass correctly 'ReverbEffect') — the create_reverb_effect-authored type is the one remaining audio asset get_audio_info doesn't recognize after the DialogueWave/DialogueVoice branches were added"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [audio, audio-authoring, reverbeffect, get_audio_info, type-unknown, readback, create-reverb-effect]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `get_audio_info` misreports a ReverbEffect's `type` as `"Unknown"`

`audio.authoring.create_reverb_effect` is a first-class authoring verb that
creates a `UReverbEffect`, but `audio.authoring.get_audio_info` on that same asset
returns `"type":"Unknown"` — even though it echoes the correct `assetClass`
`"ReverbEffect"` in the very same payload. The call succeeds and confirms nothing
about the type; the two fields contradict each other.

This is the **ReverbEffect member** of the exact `get_audio_info`
`type:"Unknown"` shape the Dialogue ticket
`E-audio-dialogue-no-readback-get-info-unknown` (IN-REVIEW) already closed for
`UDialogueWave`/`UDialogueVoice`: that fix added explicit branches so those types
return their proper `type` instead of falling through to the final `else`. The
same fix left `UReverbEffect` out — it is now the **only** `audio.authoring`
create-verb-authored asset type `get_audio_info` still doesn't recognize.

## Root cause (verified in source)

`Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`,
`get_audio_info` (`:2951-3015`) casts to `USoundCue` / `USoundWave` /
`USoundClass` / `USoundMix` / `USoundSubmix` / `USoundAttenuation` /
`UDialogueWave` / `UDialogueVoice`, and for anything else hits the final `else`
(`:3012-3015`):

```cpp
else
{
    Result->SetStringField(TEXT("type"), TEXT("Unknown"));
}
```

There is **no `UReverbEffect` branch** (`Cast<UReverbEffect>` appears nowhere in
`get_audio_info`), so a ReverbEffect — which the namespace's own
`create_reverb_effect` (`:2240-2294`) authors — falls through to `type:"Unknown"`.
The header is already included (`#include "Sound/ReverbEffect.h"`,
`:96-97`, behind `__has_include`), so a branch is cheap. The DialogueWave/
DialogueVoice branches added immediately above (`:2998-3010`, with the inline
comment "Thin recognition so get_audio_info no longer reports type:'Unknown'")
are the exact template the ReverbEffect case needs.

## Verbatim live repro (replayed against mcp__editor-automation__call)

1. `audio.authoring.create_reverb_effect { name: RE_TypeProbe, path: /Game/AudioReverbTypeProbe }`
   → `{"assetPath":"/Game/AudioReverbTypeProbe/RE_TypeProbe","message":"ReverbEffect 'RE_TypeProbe' created","assetName":"RE_TypeProbe","existsAfter":true,"assetClass":"ReverbEffect"}`
2. `audio.authoring.get_audio_info { assetPath: /Game/AudioReverbTypeProbe/RE_TypeProbe }`
   → `{"assetPath":"/Game/AudioReverbTypeProbe/RE_TypeProbe","assetClass":"ReverbEffect","type":"Unknown","message":"Audio info retrieved"}`

`assetClass` is `"ReverbEffect"` but `type` is `"Unknown"` in the same result —
an agent that authored a ReverbEffect and reads it back through the documented
verifier sees a self-contradicting type for an asset the namespace just created.

## What it should do / how to fix

Add a thin `UReverbEffect` recognition branch to `get_audio_info` (mirroring the
DialogueWave/DialogueVoice branches just above it) so it returns
`type:"ReverbEffect"` instead of `"Unknown"`. The `UReverbEffect` UPROPERTYs
(`Density`, `Diffusion`, `Gain`, `DecayTime`, `ReflectionsDelay`, `LateDelay`,
etc., engine `Sound/ReverbEffect.h`) are all `create_reverb_effect` writes a
caller might want echoed; at minimum emit `type:"ReverbEffect"` (thin recognition,
like the dialogue case), and optionally `decayTime` / `gain` to make the readback
useful. If a richer surface is wanted, a `describe_reverb_effect` reader would
match the `describe_attenuation` / `describe_sound_class` / `describe_sound_mix` /
`describe_dialogue_*` family — but the minimum bar here is just to stop reporting
`type:"Unknown"` for a type the namespace authors.

## Workaround

`assetClass:"ReverbEffect"` in the same `get_audio_info` payload already names the
true class, so an agent can read the class there; the `type` field is the
misleading one. For the reverb's actual parameters, fall back to
`asset.dump { assetPath }` → `properties.json` or `property.get` on the individual
`UReverbEffect` fields.

## History
- `#2-reverb-recognition-branch` `IN-REVIEW` developer — Added a `UReverbEffect` recognition branch to `get_audio_info` so it returns `type:"ReverbEffect"` instead of falling through to the final `else` `type:"Unknown"`, mirroring the DialogueWave/DialogueVoice branches just above it. The branch is gated `#if MCP_HAS_REVERB_EFFECT` (matching `create_reverb_effect`'s own gate and the already-included `Sound/ReverbEffect.h` at `:96-101`) and, beyond the thin type label, echoes the same float UPROPERTYs `create_reverb_effect` writes (`decayTime`/`gain`/`gainHF`/`density`/`diffusion`/`decayHFRatio`) so the readback is useful. Considered the adversarial alternative of a generic `else { type = GetClass()->GetName() }` fallback but chose the per-type branch: the recognized branches all emit type-specific enriched fields (a class-name echo can't surface reverb params), it keeps the `"Unknown"` sentinel for genuinely-unrecognized types, has a smaller blast radius for a Low ergonomic fix, and matches the board-accepted dialogue-sibling pattern. File: `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp` (branch inserted after the dialogue `#endif`, before the final `else`). Regression test: `PinWright.audio.authoring.get_audio_info.RecognizesReverbEffect` in `Source/PinWright/Private/Tests/Media/TestAudioHandlers.cpp` builds a transient `UReverbEffect`, sets distinctive `DecayTime=3.75`/`Gain=0.42`, drives the production `get_audio_info` handler, and asserts `type:"ReverbEffect"` (regresses to `"Unknown"` if reverted) plus the echoed `decayTime`/`gain`. Gated `#if MCP_TEST_HAS_REVERB`.
- `#1-initial-repro` `OPEN` reporter — REALISM-mode audio-mixing-foundation task (SFX/Weapons/Footsteps SoundClass hierarchy + CombatDuck SoundMix + CombatHall ReverbEffect; outcome ergo, culprit `audio.authoring.get_audio_info`). The attempt's friction note flagged it verbatim: "get_audio_info on the ReverbEffect reports \"type\":\"Unknown\" (assetClass is correctly ReverbEffect), a small readback-typing gap." Replay-confirmed live against mcp__editor-automation__call: `create_reverb_effect { RE_TypeProbe }` → `existsAfter:true, assetClass:ReverbEffect`, then `get_audio_info { RE_TypeProbe }` → `{"assetClass":"ReverbEffect","type":"Unknown","message":"Audio info retrieved"}` — `assetClass` correct, `type` "Unknown" in the same payload. Root cause: `get_audio_info` (`AudioAuthoringHandler.cpp:2951-3015`) has cast branches for SoundCue/Wave/Class/Mix/Submix/Attenuation/DialogueWave/DialogueVoice but NO `UReverbEffect` branch, so a ReverbEffect hits the final `else` `type:"Unknown"` (`:3012-3015`) — despite `create_reverb_effect` (`:2240`) authoring exactly that type and `Sound/ReverbEffect.h` already being included (`:96-97`). This is the ReverbEffect member of the same `get_audio_info` type:"Unknown" shape `E-audio-dialogue-no-readback-get-info-unknown` (IN-REVIEW) closed for the Dialogue types by adding recognition branches (`:2998-3010`); ReverbEffect is the one create-authored audio type that fix left out. Dedup: ripgrep over OPEN + closed found no ReverbEffect get_audio_info / type-unknown ticket; the Dialogue ticket is the nearest sibling (different asset type, MCP_HAS_DIALOGUE-gated) and the readback-thinness tickets (`E-audio-get-info-soundclass-mix-readback-thin`, `E-audio-authoring-attenuation-readback-undocumented`) are about thin-but-recognized types, not an unrecognized type:"Unknown". Distinct also from the corruption ticket filed on the same task (`B-audio-create-save-no-disk-write`, the SaveAudioAsset no-disk-write that lost all five assets on cold restart) — that is the write/persistence defect; this is the readback type-misreport.
