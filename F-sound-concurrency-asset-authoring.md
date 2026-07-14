---
id: F-sound-concurrency-asset-authoring
title: "No audio.authoring.create_sound_concurrency verb — set_cue_concurrency has no way to author the USoundConcurrency it references without python.execute"
status: IN-REVIEW
severity: Medium
category: feature
tags: [audio, sound-concurrency, authoring, missing-asset-creator]
encounters: 1
lastSeen: 2026-07-11T03:02:19.4116967+03:00
---

# No RPC to create a USoundConcurrency — `set_cue_concurrency` dead-ends without one

The `audio.authoring` namespace can point a SoundCue at a `USoundConcurrency`
group (`set_cue_concurrency`, `concurrencyPath`) and can pass a `concurrencyPath`
to runtime playback verbs (`play_sound`, `create_ambient`), but it ships **no
verb to create the `USoundConcurrency` asset those references require**. So the
shared-concurrency workflow is only half-supported: an agent can wire a cue to a
concurrency group but cannot author the group itself through the RPC surface.

The one input `set_cue_concurrency` needs can only be produced by dropping out of
the RPC surface into `python.execute` (`unreal.SoundConcurrencyFactory` +
`SoundConcurrencySettings` — set `MaxCount`, `ResolutionRule`, etc. by hand), a
documented raw-scripting escape hatch, not a supported authoring capability.

This is asymmetric: `audio.authoring` already has `create_sound_cue`,
`create_sound_class`, `create_sound_mix`, `create_sound_submix`,
`create_source_effect_chain`, `create_source_effect_preset`, and
`create_submix_effect`, but no `create_sound_concurrency`. It mirrors exactly the
already-fixed `F-audio-submix-asset-authoring` (added `create_sound_submix`) and
the in-review `F-source-effect-preset-authoring` (added
`create_source_effect_preset`): the same "one creator verb short of a working
feature" hole, here for `USoundConcurrency`.

## Evidence (this task)

Focus `audio.authoring.set_cue_concurrency`; the story asked to put three
interaction cues under one shared "manipulation SFX" concurrency limit (~3 voices,
steal oldest). To find a create verb the agent grepped the entire wiki for
`concurrency` across all namespaces (only `set_cue_concurrency` +
`play_sound`/`create_ambient` `concurrencyPath` params surfaced), scanned
`index.md` and `misc.md`, and listed the `asset.*` verbs — none create a
concurrency asset. `asset.search_assets classNames=SoundConcurrency` confirmed
zero existed to reference. This forced the `python.execute` fallback to author
`/Game/Audio/Concurrency/CG_ManipulationSFX` (`SoundConcurrencyFactory`,
`MaxCount=3`, `StopOldest`) before any of the three `set_cue_concurrency` calls
could point at a real asset.

## Guilty surface

No `USoundConcurrency` creator is registered anywhere in the handler tree — grep
of `Source/.../Handlers/Audio/` for a concurrency creator returns only
`set_cue_concurrency` (the reference-setter). Contrast the sibling create verbs
listed above, each of which `NewObject`s its asset into `/Game/Audio/...` and
returns asset verification.

severity rationale: impact=hard-blocker-with-workaround (python.execute SoundConcurrencyFactory fabrication) x reach=rare (shared-concurrency authoring is a specialized audio path, not every-session) -> Medium

**Workaround:** fabricate the `USoundConcurrency` via `python.execute`
(`unreal.SoundConcurrencyFactory` + set `SoundConcurrencySettings.MaxCount` /
`ResolutionRule`, then save), then pass its path as `concurrencyPath` to
`set_cue_concurrency`.

**Fix:** Add `audio.authoring.create_sound_concurrency(name, path?, maxCount?,
resolutionRule?, save?)` that `NewObject`s a `USoundConcurrency` into
`/Game/Audio/...` (mirroring `create_sound_submix` / `create_source_effect_preset`),
maps the common `SoundConcurrencySettings` fields (`MaxCount`, `ResolutionRule` —
StopOldest/StopFarthest/etc., `bLimitToOwner`, `VolumeScale`), and returns asset
verification. Then `create_sound_concurrency` -> `set_cue_concurrency` (xN) closes
the shared-limit flow entirely inside the RPC surface.

## History
- `#2-fix` `IN-REVIEW` developer — GO. Added `audio.authoring.create_sound_concurrency(name, path?, maxCount?, resolutionRule?, limitToOwner?, save?)` in `Handlers/Audio/AudioAuthoringHandler.cpp` — direct `NewObject<USoundConcurrency>` into `/Game/Audio/Concurrency` (mirrors DONE `create_sound_submix` / in-review `create_source_effect_preset`), maps `maxCount`->`Concurrency.MaxCount`, `resolutionRule`->`Concurrency.ResolutionRule` (case-insensitive canonical `EMaxConcurrentResolutionRule` names via new file-static `ResolveConcurrencyResolutionRule`; unknown value rejected with `INVALID_RESOLUTION_RULE`, not fake-succeeded), `limitToOwner`->`bLimitToOwner`; then save + asset verification. Registered `INVALID_RESOLUTION_RULE` in `Handlers/ErrorCodes.h`. `VolumeScale` (4th field named in ticket prose) intentionally not exposed — private member of `FSoundConcurrencySettings` with only a getter, no public setter. Closes `create_sound_concurrency` -> `set_cue_concurrency` entirely on the RPC surface; distinct from the OPEN silent-no-op setter bug `B-cue-setter-missing-asset-silent-success`. Regression gate: adopted reporter's red test `PinWright.Audio.SoundConcurrencyAuthoring` (`Tests/Media/TestSoundConcurrencyAuthoring.cpp`) — Result={Fail} pre-fix, now Result={Success}; plugin compiles clean. Severity Medium / category feature unchanged.
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the shared-manipulation-SFX concurrency task (focus `audio.authoring.set_cue_concurrency`). Capability gap distinct from the judge's silent-no-op ticket `B-cue-setter-missing-asset-silent-success`: no verb in ANY namespace creates a `USoundConcurrency`, though `set_cue_concurrency` requires one to pre-exist. Agent grepped the whole wiki (concurrency only appears as `set_cue_concurrency` + `play_sound`/`create_ambient` params), scanned index/misc, listed `asset.*`, and confirmed `search_assets classNames=SoundConcurrency` -> 0; forced a `python.execute` (`SoundConcurrencyFactory`, MaxCount=3, StopOldest) fallback to author `CG_ManipulationSFX`. Same "one creator verb short" family as the DONE `F-audio-submix-asset-authoring` and in-review `F-source-effect-preset-authoring`. Proposed: add `audio.authoring.create_sound_concurrency(name, path?, maxCount?, resolutionRule?, save?)` mirroring the other create_* audio verbs so the pickup/snap/scale shared-limit flow stays on real RPCs.
