---
id: B-play-sound-attached-not-attached
title: "audio.play_sound_attached reports success + componentPath but the AudioComponent is never attached (AttachParent null) — it does not follow the target actor as documented"
status: WONTFIX
severity: High
category: bug
tags: [audio, spawn, attach, silent-false-success, spawn-attach-parent-null]
encounters: 1
lastSeen: 2026-07-02T01:22:19.0675539+03:00
---

# audio.play_sound_attached reports success but the component is not actually attached to the target actor

The `audio.play_sound_attached` wiki page promises:

> "Play a sound from an AudioComponent **that follows a target actor** (optionally
> a socket). The component auto-destroys when playback finishes."

In practice the call returns a fully success-shaped response — a `componentPath`
nested under the target actor and `ownerActorPath` = the target actor — but the
created `UAudioComponent` is **never attached** to that actor. Its `AttachParent`
reads `null` and the target actor's root component has an **empty** `AttachChildren`.
So the audio source will **not** track / follow the prop as it moves — the entire
documented purpose of the verb ("follows a target actor") silently does not happen.

This is a silent success-with-no-effect: the caller is told (by the success shape,
the actor-owned `componentPath`, and the method's documented promise) that they now
have an attached audio source that follows the actor, but the attachment that would
make it follow was never established. The task that surfaced this explicitly wanted
to "be confident the whoosh audio source is genuinely attached to UELogo ... not
just fired and forgotten" — which is exactly the guarantee the tool fails to deliver
while reporting success.

The same defect affects the sibling **`audio.create_audio_component`** when given
`attachTo`/`actorName`: it routes through the identical
`UGameplayStatics::SpawnSoundAttached` call (`AudioHandler.cpp:918-920`) and leaves
the returned component with `AttachParent = null` as well.

## Root cause

`audio.play_sound_attached` hands the target actor's root component to
`UGameplayStatics::SpawnSoundAttached` and reports success whenever the return is
non-null, **without ever verifying the attachment took effect**:

`Plugins/PinWright/Source/PinWright/Private/Handlers/Audio/AudioHandler.cpp:499-514`

```cpp
    UAudioComponent* AudioComp = UGameplayStatics::SpawnSoundAttached(
        Sound, AttachComp, FName(*AttachPoint), FVector::ZeroVector,
        EAttachLocation::KeepRelativeOffset, true);

    TSharedPtr<FJsonObject> Resp = MakeShared<FJsonObject>();
    if (AudioComp)
    {
        Resp->SetStringField(TEXT("componentName"), AudioComp->GetName());
        AddAssetVerification(Resp, Sound);
        AddComponentVerification(Resp, AudioComp);
        Ctx.SendSuccess(Resp);
    }
```

Empirically, the `UAudioComponent` produced by `SpawnSoundAttached` in the **editor
world** comes back unattached (`AttachParent` null via two independent readback
routes; the target root's `AttachChildren` empty), so a non-null return does **not**
imply "attached and following". The engine's `SpawnSoundAttached`
(`UE_5.7/.../Private/GameplayStatics.cpp:1830-1861`) creates the component via
`FAudioDevice::CreateComponent` and only runs its attach block inside
`if (UWorld* ComponentWorld = AudioComponent->GetWorld())` — in the editor context
that attach does not stick. The handler treats the non-null pointer as proof of
success and never re-checks `AudioComp->GetAttachParent()`.

## What it should do

`audio.play_sound_attached` (and `audio.create_audio_component` with `attachTo`)
should guarantee the returned component is genuinely attached to the requested
actor/socket, or report an honest error if it can't be. Concretely:

- After `SpawnSoundAttached`, explicitly attach and verify:
  `AudioComp->AttachToComponent(AttachComp, FAttachmentTransformRules::KeepRelativeTransform, FName(*AttachPoint))`
  then confirm `AudioComp->GetAttachParent() == AttachComp` before sending success;
  if the attach did not take, send an error (e.g. `ATTACH_FAILED`) rather than a
  success shape.
- Echo the attachment in the response (`attachParent`/`attached:true`) so the caller
  can verify the follow relationship exists rather than trusting the bare
  `componentPath`.

## Verbatim repro (replayed at HEAD)

`audio.play_sound_attached`
args:
```json
{"soundPath":"Whoosh_Cue","actorName":"UELogo"}
```
result (success-shaped, component owned by UELogo):
```json
{"componentName":"AudioComponent_0","assetPath":"/Game/ExampleContent/Audio/Whoosh_Cue","assetName":"Whoosh_Cue","existsAfter":true,"assetClass":"SoundCue","componentClass":"AudioComponent","componentPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.StaticMeshActor_1.AudioComponent_0","ownerActorPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.StaticMeshActor_1","ownerMapPath":"/Game/Maps/ExampleProjectWelcome"}
```
readback #1 — `property.get` on the returned `componentPath`, `propertyName:"AttachParent"`:
```json
{"propertyName":"AttachParent","value":null,"assetName":"AudioComponent_0","assetClass":"AudioComponent"}
```
readback #2 — `system.inspect.inspect_object` on the same `componentPath` → `AttachParent`:
```json
{"AttachParent":{"type":"TObjectPtr<USceneComponent>","value":null,"is_overridden_locally":true}}
```
parent-side confirmation — `property.get` on UELogo's root `StaticMeshComponent0`, `propertyName:"AttachChildren"`:
```json
{"propertyName":"AttachChildren","value":[],"assetName":"StaticMeshComponent0","assetClass":"StaticMeshComponent"}
```

Both child-side (`AttachParent` null, via `property.get` AND `inspect_object`) and
parent-side (`AttachChildren` empty) readbacks agree: nothing is attached to UELogo,
so the "follows a target actor" contract is silently unmet.

severity rationale: impact=silent-false-success (the caller is told, by the success
shape + actor-owned componentPath + documented "follows a target actor", that they
have an attached follow-source, but AttachParent is null and the target root has no
attach children — the promised tracking never exists) × reach=normal audio path (not
every-session, but the failure is total and deterministic on the verb's sole purpose,
so not bumped down) -> High

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `audio.play_sound_attached {soundPath:"Whoosh_Cue", actorName:"UELogo"}` at HEAD: returns a success shape with a `componentPath` nested under UELogo and `ownerActorPath`=UELogo, but three independent readbacks show the component is not attached — `property.get` AttachParent = null, `inspect_object` AttachParent = null, and the target root `StaticMeshComponent0`'s `AttachChildren` = `[]`. So the documented "AudioComponent that follows a target actor" never gets its follow relationship; the verb is a silent false-success. Guilty handler line `AudioHandler.cpp:499-514` (reports success on any non-null `SpawnSoundAttached` return without verifying `GetAttachParent()`); the sibling `audio.create_audio_component` with `attachTo` shares the same `SpawnSoundAttached` path (`:918-920`) and the same AttachParent-null result. Dedup: ripgrep over OPEN/closed — `E-spawned-audio-component-not-actor-readable` (IN-REVIEW, docs) is scoped to READBACK discoverability of WorldSettings-owned location spawns and explicitly asserts the verbs "work" (only steering readback to `componentPath`); it does not cover the functional attach-null defect proven here, and its WorldSettings-owner premise does not apply to `play_sound_attached` (owner is the target actor). `B-create-ambient-sound-no-actor` (actor-vs-component) and `B-create-ambient-sound-location-object-dropped` (location shape) are different verbs/issues. No existing ticket owns the `spawn-attach-parent-null` symptom family. Filed new.
- `#2-wontfix` `WONTFIX` developer — Not-a-bug: the "component never attached / AttachParent null / silent false-success" report is a misread of the verb's documented auto-destroy lifecycle. Engine source disproves the ticket's root cause: `UGameplayStatics::SpawnSoundAttached` builds `FCreateComponentParams(ThisWorld, AttachToComponent->GetOwner())` (GameplayStatics.cpp:1820) and `FAudioDevice::CreateComponent` outers the component to the target actor via `NewObject<UAudioComponent>(Params.Actor,...)` (AudioDevice.cpp:6615), so `AudioComponent->GetWorld()` returns the editor world (non-null) and the attach block `AttachToComponent(..., KeepRelativeTransform, AttachPointName)` (GameplayStatics.cpp:1833-1843) DOES execute in the editor — the ticket's "attach only runs inside if(GetWorld()) and does not stick in editor" premise is factually wrong. (The repro's own `componentPath` nested under UELogo/StaticMeshActor_1 is the NewObject OUTER, confirming actor ownership — it is not evidence of a failed attach.) The null AttachParent on the ticket's LATER, separate readbacks is expected post-playback state: `play_sound_attached` calls the overload with `bAutoDestroy` defaulting true (AudioHandler.cpp:447 "auto-destroys when playback finishes"; :499-501 pass only the 6th arg `bStopWhenAttachedToDestroyed=true`), so after the short `Whoosh_Cue` finishes the engine `DestroyComponent()`s it and detaches — the ticket never shows AttachParent null immediately at spawn, which a genuine attach failure would require. The proposed fix (`AudioComp->AttachToComponent(AttachComp, FAttachmentTransformRules::KeepRelativeTransform, ...)`) is byte-for-byte the call the engine already makes at GameplayStatics.cpp:1843 — a behavioral no-op that would pass its own verify and still return success, adding only a misleading `attached:true` echo on a fire-and-forget component that auto-destroys moments later. The `create_audio_component` half is pure inference (no repro; its attachTo branch uses the 7-arg overload with `bStopWhenAttachedToDestroyed=false` at AudioHandler.cpp:918-920 — a different path). Lens split: adversarial=wontfix; correctness=reword but itself concedes the fix "does not deliver a durable follow" and the verb is documented transient/auto-destroy; board-historian=valid on dedup only. Objection wins: the reported defect is not established (attach genuinely occurs) and the fix duplicates an engine call — closing WONTFIX. No code.
