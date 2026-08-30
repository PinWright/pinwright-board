---
id: E-audio-authoring-attenuation-readback-undocumented
title: "audio.authoring 'Inspect-after-mutate' overlay names a readback for every audio type except SoundAttenuation — no describe, no documented asset.dump fallback"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, audio, sound-attenuation, readback, discovery, wiki, inspect-after-mutate]
---

# `audio.authoring` "Inspect-after-mutate" overlay has no readback path for SoundAttenuation

This is the **discovery/docs** half of the SoundAttenuation readback gap. The
missing live RPC itself is filed by the per-finding judge as
`F-rpc-audio-describe-attenuation` (feature). This ticket is the distinct
PROCESS angle: even before any RPC ships, an agent that follows the
`audio.authoring` overlay's own "Inspect-after-mutate" guidance after a
configure run has **no documented next step** for a SoundAttenuation asset, so
it must leave the namespace and re-navigate index pages to rediscover the
`asset.dump` workaround.

The overlay (`docs/wiki-src/audio.authoring.md`) carries a dedicated
`## Inspect-after-mutate` section that enumerates a readback per audio type:

> "Use `get_audio_info` for lightweight audio metadata, `describe_sound_cue`
> for SoundCue graph verification, `describe_metasound` for MetaSound graph
> snapshots, and `describe_sound_wave` for raw SoundWave compression / duration
> / channel metadata. The describe calls are dump-parity live reads ... while
> `get_audio_info` remains a small compatibility surface ..."

SoundCue, MetaSound, and SoundWave each get a named `describe_*` here.
**SoundAttenuation is named nowhere in this section** — and the asset has the
largest configure surface in the namespace (`create_attenuation_settings` plus
four `configure_*` mutators writing ~12 `FSoundAttenuationSettings` fields).
`get_audio_info` on a SoundAttenuation returns only `falloffDistance` +
`spatialize` (`AudioAuthoringHandler.cpp:2445-2450`), and the overlay does not
mention this thinness, does not point to a `describe_attenuation` (none exists),
and — the load-bearing miss — does **not** point to the `asset.dump` →
`properties.json` fallback that is the only live way to confirm a configured
attenuation profile today. So the section that exists to answer "how do I
verify my write?" silently omits the one audio type whose writes most need
verifying.

## Evidence (this task)

Seed `audio.authoring.configure_reverb_send`; story built
`/Game/Audio/Attenuation/ForestAmbience_Attenuation` end-to-end (all 5 configure
verbs `ok:true`, all values persisted) then asked for a readback confirm. The
configure/readback path ran clean; the friction was purely the post-`get_audio_info`
scramble. From the friction note:

> "the documented readback `get_audio_info` only returns falloffDistance and
> spatialize ... so it could not confirm the reverb/occlusion/algorithm fields;
> I had to fall back to `asset.dump` ... and read `properties.json` to verify
> the writes stuck. **One extra namespace-index lookup to find that read path.**"

The call log shows the cost concretely: after the six intended calls, the agent
emitted four extra discovery/read calls to recover —
`call("audio.authoring")` (namespace index, "find describe/dump read"),
`call("asset")` (namespace index), `call("asset.dump")` (wiki-nav), then the
`asset.dump` execute. Two of those are pure index re-navigation that a single
line on the overlay would have eliminated. The describe family is right there in
the same section; an agent reasonably expects an `audio.authoring` answer and
spends calls confirming there isn't one before pivoting to the `asset`
namespace.

This is the same shape as `E-game-framework-info-not-asset-readback` (OPEN,
`docs`-tagged): a namespace's documented "did it work?" read can't confirm the
asset its write verbs configure, the overlay never flags the asymmetry, and the
agent burns extra calls discovering the real readback path. There the cheap win
was documenting the supported confirm path on the overlay; same here.

## What it should do / how to fix (docs-first, NAMES the overlay page)

Improve `docs/wiki-src/audio.authoring.md`, `## Inspect-after-mutate` section.
The edit is a downstream wiki process, not this audit's job — naming the page
and the change is the deliverable:

- Add SoundAttenuation to the readback enumeration. Until/unless
  `describe_attenuation` ships (`F-rpc-audio-describe-attenuation`), state
  plainly that `get_audio_info` on a SoundAttenuation returns only
  `falloffDistance` + `spatialize`, and that the full configured profile
  (`distanceAlgorithm`, `RadiusMin`, `spatializationAlgorithm`, and the reverb /
  occlusion blocks the four `configure_*` verbs write) is confirmed via
  `asset.dump { assetPath }` → `properties.json` (the `FSoundAttenuationSettings`
  struct lives under the `Attenuation` UPROPERTY). When `describe_attenuation`
  lands, swap the line to name it (the `F-` ticket's fix block already plans to
  update this same section — this ticket asks for the interim documented
  fallback so the gap is closed for callers *now*, independent of the RPC).

This is doc-only and self-contained; it removes the namespace-index
re-navigation regardless of whether the RPC is ever built.

**Workaround:** confirm a configured attenuation profile via
`asset.dump { assetPath }` → `properties.json`, not `get_audio_info`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a 6-intended-call SoundAttenuation build (seed `audio.authoring.configure_reverb_send`; all 5 configure verbs `ok:true`, all values persisted). Outcome/feature gap filed separately by the judge as `F-rpc-audio-describe-attenuation` (missing live RPC). Distinct PROCESS angle: the `audio.authoring` overlay's `## Inspect-after-mutate` section names a readback for SoundCue/MetaSound/SoundWave but **none for SoundAttenuation** and never documents the `get_audio_info` thinness or the `asset.dump`→`properties.json` fallback. Friction note: "one extra namespace-index lookup to find that read path"; call log confirms 4 recovery calls (`audio.authoring` index, `asset` index, `asset.dump` wiki-nav, `asset.dump` execute), two of them pure index re-navigation a single overlay line would remove. Same docs-discoverability shape as `E-game-framework-info-not-asset-readback`. Proposed: add a SoundAttenuation readback line to `docs/wiki-src/audio.authoring.md` `## Inspect-after-mutate` pointing at `asset.dump`→`properties.json` now, swapping to `describe_attenuation` if/when `F-rpc-audio-describe-attenuation` ships. Culprit method: `audio.authoring.get_audio_info` (the thin read the overlay implies is sufficient); configure verbs all work.
- `#2-additional-short-falloff-attenuation` `OPEN` reporter — Additional evidence (audio.authoring SoundClass-tree + StealthDuck mix + short-falloff ambient build, outcome tool_bug for the separate looping no-op judge-filed `B-create-sound-cue-looping-noop`). Same SoundAttenuation readback-thinness shape, here on the **inner radius**: story step 5 asked for `ATT_AmbientShort` with `inner radius ~400, falloff ~2000` and step 7 to "confirm the attenuation reference … took". After `configure_distance_attenuation { inner400 falloff2000 NaturalSound }`, `get_audio_info { ATT_AmbientShort }` echoed `falloffDistance`/`spatialize` only — NOT the inner radius the story explicitly set — so the agent could not confirm the inner-radius write through the documented reader and pivoted to `property.get` to read `AttenuationShapeExtents.X = 400` (plus separate `property.get` calls for `FalloffDistance`=2000 and `DistanceAlgorithm`=NaturalSound). Friction note (verbatim): "get_audio_info on the attenuation only echoes falloffDistance/spatialize (not the inner radius) … so I fell back to property.get to confirm AttenuationShapeExtents.X=400". A `property.get` on the whole `Attenuation` struct returned empty output (the agent had to address the leaf `AttenuationShapeExtents` field directly) — extra friction on the fallback path itself. Confirms the `get_audio_info` SoundAttenuation thinness is unchanged and that even the documented-elsewhere `property.get` workaround needs leaf-field addressing, not the struct. No re-file; folded here.
- `#3-already-fixed` `IN-REVIEW` developer — Already resolved in current source; no code/doc change needed. This ticket's sole deliverable (add a SoundAttenuation readback line to `Docs/wiki-src/audio.authoring.md` `## Inspect-after-mutate`, pointing at `asset.dump`→`properties.json` now / `describe_attenuation` later) is fully done — and via the superior outcome (the live RPC), not the interim `asset.dump` fallback. The sibling `F-rpc-audio-describe-attenuation` (status IN-REVIEW) landed both halves: (1) the live RPC `audio.authoring.describe_attenuation` is registered at `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp:2518` (loads via `LoadSoundAttenuationFromPath`, returns `ATTENUATION_NOT_FOUND` on failure, hand-rolls the full configure surface — distanceAlgorithm/falloffDistance/innerRadius=AttenuationShapeExtents.X/spatialize/spatializationAlgorithm/occlusion block/reverb-send block), with regression tests at `Private/Tests/Media/TestAudioHandlers.cpp:998` (`.EchoesConfiguredFields`) and `:1097` (`.MissingRequiredParam`); (2) the overlay `Docs/wiki-src/audio.authoring.md:7` `## Inspect-after-mutate` line now names `describe_attenuation` "for the full SoundAttenuation surface the four `configure_*` verbs write", and a dedicated H3 at `:75-77` states verbatim that `get_audio_info` returns only `falloffDistance`+`spatialize` for a SoundAttenuation and to "use `describe_attenuation` to confirm what the four `configure_*` verbs wrote without falling back to `asset.dump`'s `properties.json`" — exactly the enumeration gap + thinness note this ticket requested. The ticket's stale cite (`AudioAuthoringHandler.cpp:2445-2450`) and its "describe_attenuation (none exists)" premise are both outdated (the get_audio_info SoundAttenuation branch is now at `:2658-2663` and is the intentional lightweight summary, backstopped by `describe_attenuation`). Released the fuzz2 claim. Flipping OPEN→IN-REVIEW for a tester to verify the overlay/RPC are present; no separate fix to make.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 1 citation sits in history rows and is left verbatim per the append-only rule. The one citation is in history row `#3` and stays verbatim. Map: `Handlers/Audio/AudioAuthoringHandler.cpp:2518` → `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp:2953` (the `audio.authoring.describe_attenuation` `REGISTER_RPC_HANDLER`; section banner `:2943`). `:2518` at HEAD is a `RequireString` inside a reverb-effect create handler. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
