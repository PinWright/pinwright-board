---
id: B-create-ambient-sound-location-object-dropped
title: "audio.create_ambient_sound silently drops an object-form {x,y,z} location and spawns the actor at origin"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, ambient-sound, location, silent-no-effect, param-shape]
---

# audio.create_ambient_sound silently drops an object-form {x,y,z} location and spawns the actor at origin

`audio.create_ambient_sound` documents `location` as an **array** `[X,Y,Z]`,
while its sibling `audio.create_audio_component` documents `location` as an
**object** `{x,y,z}`. The two verbs sit in the same namespace and do the same
conceptual thing (place a positioned audio source in the world), so passing the
object form to `create_ambient_sound` is a natural mistake — especially since
`create_audio_component` accepts BOTH shapes.

When `audio.create_ambient_sound` is given an **object** `location`
(`{"x":111,"y":222,"z":333}`), it returns a fully success-shaped response
(`existsAfter:true`, `actorClass:"AmbientSound"`, an `actorPath`/`componentPath`)
with **no error and no warning** — but the location is **silently discarded**
and the spawned actor's `UAudioComponent` ends up at the world origin
`[0,0,0]` instead of the requested position. The documented array form
`[111,222,333]` places it correctly at `[111,222,333]`.

This is a silent success-with-no-effect: the caller is told the ambient source
was placed where they asked, but it is actually at the origin. There is no
diagnostic to surface the dropped parameter, so the misplacement is only
discoverable by reading `RelativeLocation` back.

Two reasonable fixes (either resolves it):
- **Coerce both shapes.** Accept an object `{x,y,z}` location in
  `create_ambient_sound` just as `create_audio_component` already does, so the
  two siblings are symmetric.
- **Reject the unknown shape.** If only the array form is supported, return a
  clean `[UNKNOWN_PARAMS]`/validation error when `location` is not an array,
  instead of silently dropping it.

(Aligning the two verbs on a single documented location shape would also remove
the cross-verb inconsistency the attempt agent tripped on.)

**Workaround:** Pass `location` as an array `[X,Y,Z]` to
`audio.create_ambient_sound` (the documented form); the object form is silently
ignored.

**Fix:** In `audio.create_ambient_sound`'s arg parsing, parse the `location`
object form (or validate-and-reject non-array input) rather than letting an
unrecognized shape fall through to a default-constructed zero vector.

## Verbatim repro

`audio.create_ambient_sound` — object-form location (silently dropped):
args:
```json
{"soundPath":"/Engine/EditorSounds/GamePreview/StartPlayInEditor_Cue.StartPlayInEditor_Cue","location":{"x":111,"y":222,"z":333}}
```
result (success-shaped):
```json
{"actorPath":"/Game/Maps/ExampleProjectWelcome","actorName":"AmbientSound2","actorClass":"AmbientSound","componentPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.AmbientSound_2.AudioComponent0","componentName":"AudioComponent0"}
```
readback — `property.get` on that `componentPath`, `propertyName:"RelativeLocation"`:
```json
{"propertyName":"RelativeLocation","value":[0,0,0]}
```

`audio.create_ambient_sound` — documented array-form location (placed correctly):
args:
```json
{"soundPath":"/Engine/EditorSounds/GamePreview/StartPlayInEditor_Cue.StartPlayInEditor_Cue","location":[111,222,333]}
```
readback — `property.get` on the resulting `componentPath`, `propertyName:"RelativeLocation"`:
```json
{"propertyName":"RelativeLocation","value":[111,222,333]}
```

For contrast, the sibling `audio.create_audio_component` (documented `location`
as `{x,y,z}`) accepts an **array** `[800,0,100]` and applies it correctly
(`RelativeLocation` = `[800,0,100]`), so the two verbs disagree on which shapes
they tolerate.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `audio.create_ambient_sound` with an object-form `location` `{x:111,y:222,z:333}`: returns success (`actorClass:"AmbientSound"`, a real actor + componentPath) but `property.get RelativeLocation` on the spawned component reads `[0,0,0]` — the location is silently dropped and the actor lands at the world origin. The documented array form `[111,222,333]` places it correctly at `[111,222,333]`. Sibling `audio.create_audio_component` (docs say object `{x,y,z}`) accepts an array `[800,0,100]` and applies it, so the two verbs tolerate different location shapes; the object form on `create_ambient_sound` is a natural mistake that fails silently. Distinct from `B-create-ambient-sound-no-actor` (actor-vs-component, now IN-REVIEW/fixed — the actor IS spawned here).
- `#2-coerce-both-shapes` `IN-REVIEW` developer — Replaced the array-only `TryGetArrayField` `location` parse in `audio.create_ambient_sound` with the shared `ExtractVectorField(RawPayload, TEXT("location"), FVector::ZeroVector)` helper, so object `{x,y,z}` and array `[X,Y,Z]` are both coerced — identical to the sibling `audio.create_audio_component` (which already routes through `ExtractVectorField`), removing the cross-verb asymmetry. Also widened the param-schema doc string from `World location [X,Y,Z]` to note both shapes. File: `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioHandler.cpp` (handler block ~628 and param spec line ~610). Regression test `FExtractVectorFieldAmbientSoundObjectLocationTest` (`EditorAutomationRpcGateway.core.json.extract_vector_field.AmbientSoundObjectLocation`) in `Source/.../Private/Tests/Core/TestJsonUtils.cpp` feeds the verbatim ticket payload `{x:111,y:222,z:333}` through the production `ExtractVectorField` path the handler now uses and asserts it resolves to `[111,222,333]` and is not nearly-zero (would have read origin under the old array-only parse). Did not compile/run (later phase verifies).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
