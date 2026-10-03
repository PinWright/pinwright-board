---
id: B-audio-play-verbs-ignore-pie-world
title: "audio.play_sound_at_location returns success during PIE but plays into the editor world, so nothing is audible or listed; its description also claims a transient AudioComponent that PlaySoundAtLocation never creates"
status: OPEN
severity: Medium
category: bug
tags: [audio, runtime, pie, world-resolution, silent-success, docs, play_sound_at_location, play_sound_2d]
encounters: 1
lastSeen: 2026-10-02T18:45:00Z
rice: [1, 3, 1, 1]
priority: 33
---

# Runtime audio play verbs always target the editor world

`audio.play_sound_at_location` resolves its world with
`GEditor->GetEditorWorldContext().World()` (`Handlers/Audio/AudioHandler.cpp:248`) and
`audio.play_sound_2d` does the same (`AudioHandler.cpp:298`). There is no PIE branch and no
`world` parameter. The handler description says "Runtime op (works in PIE/editor world)".

## Observed (wt2, Linux, UE 5.8, offscreen editor with a real audio device)

With a PIE session running (`editor.play {}` -> `pieActive:true`):

```
audio.play_sound_at_location {"soundPath":"/Game/Vefects/Free_Fire/Shared/Audio/SFX_FireBig_L","location":[10,20,30]}
-> {"success":true,"soundPath":"/Game/Vefects/Free_Fire/Shared/Audio/SFX_FireBig_L","location":{"x":10,"y":20,"z":30}}

audio.list_active_sounds {}
-> {"devicesInspected":2,"count":1,...,"sounds":[ LobbyMusic_Cue on deviceId 2, worldType "PIE" ]}
```

`SFX_FireBig_L` is a looping wave, so the device would have kept it in either the active list or
the virtual-loop map. It is in neither, on either device. The call reported success and did
nothing. The same asset class played through `python.execute` with
`UGameplayStatics.play_sound_at_location(<PIE world>, ...)` in the same PIE session is listed
immediately (`worldType:"PIE"`, `audioComponentPath:null`), so the device and the asset are fine;
only the verb's world choice is wrong.

Outside PIE the verb works: the sound lands on the editor device (`worldType:"Editor"`).

## Second defect: the description is wrong about the component

The description and wiki page say the verb "spawns a transient AudioComponent that auto-destroys
when finished". It calls `UGameplayStatics::PlaySoundAtLocation`, which creates no component;
`audio.list_active_sounds` on the result shows `audioComponentPath:null, audioComponentId:0`.
A caller following the description would look for a component that does not exist, which is the
exact trap `F-audio-no-active-sound-enumeration` was filed about.

## Ask

- Resolve the PIE world when PIE is active (the way other runtime verbs do), or take an explicit
  `world` parameter; never report success for a play the active session cannot hear. Echo the
  world used in the response.
- Fix the `play_sound_at_location` description: fire-and-forget, no component.

## Scope note

Only `play_sound_at_location` was exercised during PIE. `play_sound_2d`, the sound-mix verbs
(`push_sound_mix`, `pop_sound_mix`, `set_sound_mix_class_override`), and the actor-targeting verbs
(`play_sound_attached`, `fade_sound_in/out` via `FindAudioActorByName`) read the same
`GetEditorWorldContext().World()` pattern (18 call sites in `Handlers/Audio/`) and are likely
affected the same way; unverified.

## Severity

Silent false-success is High by impact class; the runtime audio play verbs during PIE are a rare
path, so one level down: Medium.

## History
- `#1-filed` `OPEN` G16-live-tester — Found during the live check of `audio.list_active_sounds` (`F-audio-no-active-sound-enumeration`). During PIE, `audio.play_sound_at_location` on looping `SFX_FireBig_L` returned `success:true` but no row appeared on either audio device; the handler plays into `GEditor->GetEditorWorldContext().World()` (`AudioHandler.cpp:248`; `play_sound_2d` same at `:298`). The same play through `python.execute` into the PIE world is listed at once. Also: the verb's description claims a transient AudioComponent, while `list_active_sounds` shows `audioComponentPath:null` for its sounds. Dedup: no board ticket mentions PIE-world targeting for audio verbs; `B-play-sound-attached-not-attached` (WONTFIX) and `E-spawned-audio-component-not-actor-readable` are about component attachment/readback, not world choice.
