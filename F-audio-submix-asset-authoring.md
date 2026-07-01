---
id: F-audio-submix-asset-authoring
title: "Submix routing gaps: no ParentSubmix wiring, no USoundClass output target, and existing creator is misnamed"
status: DONE
severity: Medium
category: feature
tags: [audio, submix, sound-class, routing, authoring, naming]
---

# Submix routing gaps: no ParentSubmix wiring, no USoundClass output target, and existing creator is misnamed

The `audio.authoring` namespace can fabricate the asset nodes of a mix
hierarchy (`create_sound_class`, `create_sound_mix`, the existing
`create_submix_effect`) and tweak per-class scalars
(`set_class_properties`, `set_class_parent`), but it cannot wire the
submix graph itself. That leaves the namespace one verb short of a
working master-bus / sub-bus topology, which is the first thing any
real audio mix authoring agent needs to build.

Three concrete gaps, verified against
`Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`:

1. **`create_submix_effect` is misnamed and under-spec'd.** Lines
   2264–2320 register `audio.authoring.create_submix_effect` with the
   summary "Create a SoundSubmix asset". It calls
   `NewObject<USoundSubmix>` and ignores the declared `effectType`
   parameter entirely. It is *not* a submix-effect-preset creator
   (`USoundEffectSubmixPreset`); it is the submix-asset creator that
   the proposal asked for. The name mislabels what it does, the
   parameter `effectType` is dead, and it never sets a parent submix
   even when one is desired at creation time. An agent reading the
   namespace index will assume an effect-preset RPC exists and that
   the submix-asset RPC is missing.

2. **No way to set `USoundSubmix::ParentSubmix` after creation.** Grep
   for `ParentSubmix` in `Private/Handlers/Audio/` returns zero hits.
   There is no `audio.authoring.set_submix_parent` (or equivalent
   field on `create_submix_effect`). A two-level bus tree (master →
   SFX/Music/UI) is unbuildable without re-saving the asset by hand.

3. **`set_class_properties` does not expose
   `USoundClass::Properties.ParentSubmix`.** The handler at lines
   1269–1316 covers `Volume`, `Pitch`, `LowPassFilterFrequency`,
   `LFEBleed`, `VoiceCenterChannelVolume`, but not `ParentSubmix`,
   which is the field that routes a sound class's mixed output into
   a specific submix. Without it, classes always route to the
   project's master submix regardless of the bus tree we just built.

The rest of the namespace is conspicuously complete (SoundClass parent
hierarchy via `set_class_parent`, SoundMix adjusters via
`create_sound_mix` + `add_class_to_mix`, runtime activation via
`audio.push_sound_mix`), so this gap is the routing layer between
"assets exist" and "assets are connected".

**Fix:** Treat as a small audio-routing patch on the existing handler:

1. Rename + repurpose `audio.authoring.create_submix_effect` so its
   name matches what it does. Two options, choose one:
   - **Preferred:** add `audio.authoring.create_sound_submix` as the
     canonical name (mirrors `create_sound_class` / `create_sound_mix`)
     pointing at the same code path, keep `create_submix_effect` as
     a deprecated alias that resolves to the same handler, drop the
     dead `effectType` param from the new spec, and add an optional
     `parentSubmix` (asset path) that wires `ParentSubmix` at creation.
     Update the summary on the alias to flag deprecation.
   - **Alternative:** keep the existing name and just add the
     `parentSubmix` param. Cheaper but leaves the misnomer in the
     public namespace forever.
2. Add `audio.authoring.set_submix_parent(assetPath, parentPath?,
   save?)` — loads the target `USoundSubmix`, assigns
   `ParentSubmix` (or `nullptr` when omitted), `Modify()` +
   `MarkPackageDirty()` + optional save, returns asset verification.
   Mirror `set_class_parent` exactly for API symmetry.
3. Extend `set_class_properties` with an optional `parentSubmix`
   (string, asset path) that, when present, resolves to a
   `USoundSubmix*` and assigns `SoundClass->Properties.ParentSubmix`.
   Keep all current fields backward compatible — only act when the
   key is present, mirroring the existing per-field conditional
   pattern in that handler.
4. Add asset-dump readback parity: when a `USoundSubmix` is dumped,
   include `parentSubmix` (path) so verification round-trips through
   the cache. The existing `USoundClass` branch at lines 2431–2440
   should also emit `outputSubmix` from `Properties.ParentSubmix`.
5. Tests under `Source/EditorAutomationRpcGatewayTests/` covering:
   create-parent → create-child-with-parentSubmix → read back via
   asset verification; `set_submix_parent` to null clears the
   parent; `set_class_properties` with `parentSubmix` writes the
   field and round-trips through the dump.

Broad UE audience: any project shipping with bus-routing
(SFX/Music/UI/Voice splits — i.e. any non-trivial title) needs this.

## History
- `#1-three-routing-gaps` `OPEN` reporter — Verified against `AudioAuthoringHandler.cpp`: `create_submix_effect` (lines 2264–2320) is the de-facto `create_sound_submix` (creates `USoundSubmix`, ignores its declared `effectType` param, never sets a parent); no `set_submix_parent` exists (grep `ParentSubmix` in `Private/Handlers/Audio/` returns zero hits); `set_class_properties` (lines 1269–1316) omits `USoundClass::Properties.ParentSubmix`. Net effect: agents can create submix and sound-class assets but cannot wire the routing graph — output always lands on the project master. Original proposal slug retained; reshaped to (a) flag the misnamed existing creator and propose `create_sound_submix` as canonical name + `parentSubmix` param, (b) add `audio.authoring.set_submix_parent` mirroring `set_class_parent`, (c) extend `set_class_properties` with `parentSubmix`, (d) add dump readback parity for `parentSubmix` / `outputSubmix`.
- `#2-submix-routing-wired` `IN-REVIEW` developer — Added canonical create_sound_submix RPC (factored shared CreateSoundSubmixAsset helper); converted create_submix_effect into a deprecated wrapper calling the same helper. Added set_submix_parent mirroring set_class_parent. Extended set_class_properties with optional parentSubmix. get_audio_info now emits outputSubmix for USoundClass and a new USoundSubmix branch emitting parentSubmix. All submix code wrapped in #if MCP_HAS_SUBMIX. Regression test FAudioSubmixRoutingTest exercises creation-with-parent, set-then-clear, SoundClass routing, and dump readback round-trip.
- `#3-verify-routing-wired` `DONE` tester — Verified: wiki.get on audio.authoring.create_sound_submix and audio.authoring.set_submix_parent both return populated schemas. Created SM_McpVerifyParent + SM_McpVerifyChild via create_sound_submix (child with parentSubmix arg) — both succeeded. set_submix_parent cleared then re-set parent — both calls returned "Submix parent updated". asset.dump readback of child shows properties.json with `ParentSubmix.value` = "/Game/App/UI/Test/SM_McpVerifyParent_FAudioSubmix.SM_McpVerifyParent_FAudioSubmix" after re-set. Temp assets cleaned up via asset.delete.
