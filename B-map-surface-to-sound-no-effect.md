---
id: B-map-surface-to-sound-no-effect
title: "CharacterHandler footstep/movement verbs echo inputs but never persist variable defaults (map_surface_to_sound, configure_footstep_fx, add_custom_movement_mode; SetBPVarDefaultValue is itself a no-op stub)"
status: IN-REVIEW
severity: High
category: bug
tags: [character, footstep, surface, silent-noop, success-no-effect, echoes-input, map-variable, default-value]
---

# CharacterHandler footstep/movement verbs are silent no-ops that echo their inputs as if defaults were persisted

Three `CharacterHandler.cpp` verbs add empty Blueprint member variables and then
echo their input parameters in the success payload as though the values were
stored, but they never write the variable defaults. The shared root cause is the
file-local helper `SetBPVarDefaultValue` (`CharacterHandler.cpp:47-52`), which is
itself a **no-op stub** — it only `UE_LOG`s "Set default value in Blueprint
editor if needed" and writes nothing.

**1. `character.map_surface_to_sound`** (~lines 626-667) is documented as "Map a
physical surface type to a footstep sound" and accepts `surfaceType`,
`footstepSoundPath`, `footstepParticlePath`, `footstepDecalPath`. Its only side
effect is adding a single empty member variable named `FootstepSoundMap` of type
`Map<Name, SoftObject>` via `AddBlueprintVariableChar`. The four params are read
purely to **echo them in the result** — never inserted into that map, never
written to the CDO, never stored anywhere:

```cpp
FString SurfaceType = Ctx.GetString(TEXT("surfaceType"));
FString SoundPath = Ctx.GetString(TEXT("footstepSoundPath"));
// ...
AddBlueprintVariableChar(Blueprint, TEXT("FootstepSoundMap"), MapPinType, TEXT("Footsteps")); // empty map, no entries
// ...
Result->SetStringField(TEXT("surfaceType"), SurfaceType);     // echoed input
if (!SoundPath.IsEmpty()) Result->SetStringField(TEXT("sound"), SoundPath); // echoed input
Result->SetStringField(TEXT("mapVariable"), TEXT("FootstepSoundMap"));
```

So every call (re)adds the same single empty `FootstepSoundMap`; the
`surfaceType` key — the actual association the user cares about — is silently
dropped, and there is no way to build the documented surface→sound mapping at
all. The result misreports `"sound"`/`"surfaceType"` as though that pair is now
stored, when the persisted map is empty.

**2. `character.configure_footstep_fx`** (~lines 672-706) has the same shape: it
adds empty `FootstepVolumeMultiplier` / `FootstepParticleScale` float vars and
echoes `volumeMultiplier`/`particleScale` in the result, but never writes those
values into the variable defaults — so `0.8`/`1.25` are not stored either.

**3. `character.add_custom_movement_mode`** (~lines 511-575) is collateral of the
same root cause: it *does* call `SetBPVarDefaultValue` (lines 551-552) to
"persist" `modeId`/`customSpeed` into the `CustomModeId_<Mode>` / `<Mode>Speed`
vars, and echoes `modeId`/`customSpeed` in the result — but because the helper is
a no-op stub, those defaults are silently dropped too. (It separately writes
`MaxCustomMovementSpeed` on the CDO, which is fine; only the var defaults are
lost.)

## What it should do
Make `SetBPVarDefaultValue` actually persist the variable default (write the
`FBPVariableDescription.DefaultValue` string in the engine's import-text format),
and call it from `map_surface_to_sound` and `configure_footstep_fx` so a
post-compile read-back reflects the stored values. Fixing the one shared helper
repairs `add_custom_movement_mode` for free. For the surface map, accumulate
entries into the existing `FootstepSoundMap` default (so repeated calls build the
mapping) rather than overwriting an empty map each time.

**Fix:** Replace the `SetBPVarDefaultValue` stub with a real implementation that
locates the variable in `Blueprint->NewVariables` and sets its `DefaultValue`
string (using the engine import-text container format for the map type), then
`MarkBlueprintAsStructurallyModified`. Wire it into `map_surface_to_sound`
(building/extending the `FootstepSoundMap` default with the `surfaceType →
footstepSoundPath` entry) and `configure_footstep_fx` (writing the two float
defaults). `add_custom_movement_mode` already calls the helper, so it is fixed by
the helper change.

## Verbatim repro
1. `blueprint.create` `{name:"BP_SurfaceWalker_Replay", savePath:"/Game/FuzzVeh", parentClass:"Character", waitForCompletion:true}` → ok.
2. `character.map_surface_to_sound` `{blueprintPath:"/Game/FuzzVeh/BP_SurfaceWalker_Replay", surfaceType:"SurfaceType_Default", footstepSoundPath:"/Game/ExampleContent/Audio/Audio/Surfaces/Audio_Footstep_Wood01"}`
   → `{"surfaceType":"SurfaceType_Default","sound":".../Audio_Footstep_Wood01","mapVariable":"FootstepSoundMap"}`
3. `character.map_surface_to_sound` `{... surfaceType:"SurfaceType1", footstepSoundPath:".../Audio_Footstep_Glass01"}`
   → `{"surfaceType":"SurfaceType1","sound":".../Audio_Footstep_Glass01","mapVariable":"FootstepSoundMap"}`
4. `character.configure_footstep_fx` `{... volumeMultiplier:0.8, particleScale:1.25}` → `{"volumeMultiplier":0.8,"particleScale":1.25}`
5. `blueprint.compile` `{blueprintPath:".../BP_SurfaceWalker_Replay", saveAfterCompile:true}` → compiled, saved.
6. `blueprint.get` `{blueprintPath:".../BP_SurfaceWalker_Replay"}` →
   variables = `FootstepSoundMap` (Map<name,softobject>), `FootstepVolumeMultiplier` (float), `FootstepParticleScale` (float);
   **`"defaults":{}`** — the map has **no entries** and the floats have **no stored default**. Neither of the two surface→sound pairs from steps 2-3, nor the 0.8/1.25 from step 4, persisted.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed deterministically: two `map_surface_to_sound` calls (Wood01 + Glass01) and one `configure_footstep_fx` all returned success echoing their inputs, but a post-compile `blueprint.get` shows only empty `FootstepSoundMap`/`FootstepVolumeMultiplier`/`FootstepParticleScale` vars with `defaults:{}` — zero surface mappings stored, scalars not stored. Silent success-with-no-effect plus an output-field misreport (`surfaceType`/`sound` echoed as if persisted).
- `#2-reword-and-fix-shared-stub` `IN-REVIEW` developer — Reworded to its true (wider) scope: the shared root cause is the file-local `SetBPVarDefaultValue` stub, so `add_custom_movement_mode` (which already calls it) was also dropping `modeId`/`customSpeed` defaults. Fix in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Character/CharacterHandler.cpp`: replaced the no-op `SetBPVarDefaultValue` with a real implementation that writes `FBPVariableDescription.DefaultValue` (engine import-text format) on the matching `NewVariables` entry + `MarkBlueprintAsStructurallyModified`, and added a `GetBPVarDefaultValue` reader and a `BuildNameSoftObjectMapDefault` helper that upserts `surfaceType -> footstepSoundPath` into the `Map<Name,SoftObject>` default in the engine's `((Key, "Value"))` format (accumulating across repeated calls). Wired `configure_footstep_fx` to persist the two float defaults and `map_surface_to_sound` to persist/accumulate the map entry (now returns `INVALID_PARAMS` if surfaceType/sound are missing instead of a fake-success echo; result adds `entryStored`/`mapDefault`). `add_custom_movement_mode` is fixed transitively by the helper change. Regression test: `EditorAutomationRpcGateway.character.footstep_defaults.Persisted` in `Private/Tests/Gameplay/TestCharacterHandlers.cpp` creates a real ACharacter blueprint, invokes the production handlers, and asserts the persisted `FootstepVolumeMultiplier`/`FootstepParticleScale`/`FootstepSoundMap` defaults carry the supplied scalars and both accumulated surface->sound entries — fails if the stub fix is reverted. Not compiled/tested here (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
