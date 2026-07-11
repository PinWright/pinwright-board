---
id: B-configure-sprint-speed-var-unset
title: "character.configure_sprint creates a SprintSpeed blueprint variable but never writes the passed sprintSpeed to it — the variable's default stays 0"
status: IN-REVIEW
severity: Medium
category: bug
tags: [character, configure-sprint, sprint-speed, state-var-default-unset, silent-wrong-data, blueprint]
encounters: 1
lastSeen: 2026-07-11T08:55:56.1616498+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T21:12:19.0203566+03:00
---

# `character.configure_sprint` creates a `SprintSpeed` variable but leaves its default at 0

`character.configure_sprint {blueprintPath, sprintSpeed}` is documented as
"Configure sprint parameters with state variables". It creates two member
variables on the character blueprint — `bIsSprinting` (bool) and `SprintSpeed`
(float, DisplayName "Sprint Speed") — and writes the passed speed onto the
movement CDO as `MaxCustomMovementSpeed`. The problem: the `sprintSpeed` value
the caller passes is **never written to the `SprintSpeed` variable it just
created**. That variable keeps the float zero default, so a caller who passes
`sprintSpeed:900` ends up with a blueprint variable literally named "Sprint
Speed" whose default is `0`.

The whole point of a `SprintSpeed` state variable is for the character's own
blueprint graph to read it (the canonical UE sprint pattern: "on sprint pressed,
set MaxWalkSpeed = SprintSpeed"). Left at `0`, any graph logic that references it
sprints at zero speed — silent wrong data on the variable the method exists to
populate. The configured speed IS recoverable (it lands on `MaxCustomMovementSpeed`,
read back by `get_character_info` as `customMovementSpeed`), so this is a partial
failure, not a total no-op — but the created `SprintSpeed` variable is a lie.

The response echoes `sprintSpeed:900` (from the input arg, not read back off the
variable) and `stateVariable:"bIsSprinting"`, so there is no in-band signal that
the `SprintSpeed` variable was left at 0.

## Source confirmation (ground truth)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Character/CharacterHandler.cpp`,
`character.configure_sprint` (L1219-1263). The handler creates `SprintSpeed` but
never calls `SetBPVarDefaultValue` on it — the passed speed only goes to the CDO:

```cpp
// L1244 — SprintSpeed variable is created...
AddBlueprintVariableChar(Blueprint, TEXT("SprintSpeed"), FloatPinType, TEXT("Sprint"));
...
// L1252 — ...but the value only lands on MaxCustomMovementSpeed; SprintSpeed's default is never set
CharCDO->GetCharacterMovement()->MaxCustomMovementSpeed = static_cast<float>(SprintSpeed);
```

Contrast the sibling `character.add_custom_movement_mode` (same file, L701-708),
which DOES populate its `<Mode>Speed` variable via `SetBPVarDefaultValue`:

```cpp
AddBlueprintVariableChar(Blueprint, SpeedVarName, FloatPinType, TEXT("Movement States"));
SetBPVarDefaultValue(Blueprint, FName(*ModeIdVarName), FString::FromInt(ModeId));
SetBPVarDefaultValue(Blueprint, FName(*SpeedVarName), FString::SanitizeFloat(CustomSpeed));
```

The footstep handlers in the same file (`FootstepVolumeMultiplier`/
`FootstepParticleScale` at L873-880) also set their created variables' defaults.
So the omission is confined to `configure_sprint` — its siblings that create a
value variable all populate it; `configure_sprint` alone forgets.

## What it should do

After creating the `SprintSpeed` variable, set its default to the passed value —
add `SetBPVarDefaultValue(Blueprint, TEXT("SprintSpeed"), FString::SanitizeFloat(static_cast<float>(SprintSpeed)))`
(mirroring `add_custom_movement_mode`'s `<Mode>Speed` write) so the variable the
method creates actually holds the sprint speed the caller configured. Optionally
echo the applied `SprintSpeed` variable default in the result so a dropped write
is visible in-band.

severity rationale: impact=silent wrong data on a created variable (SprintSpeed=0 vs the configured 900; a graph reading it breaks), softened because the value DOES land on MaxCustomMovementSpeed and is recoverable via get_character_info × reach=character sprint setup (moderate, not every-session) -> Medium.

## Verbatim repro (replayed live against `mcp__pinwright__call`)

1. `blueprint.create {name:"BP_OracleSprintReplay", savePath:"/Game/OracleReplay", parentClass:"Character"}` → `{"path":"/Game/OracleReplay/BP_OracleSprintReplay","saved":true,"existsAfter":true,"assetClass":"Blueprint"}`.
2. `character.configure_sprint {blueprintPath:"/Game/OracleReplay/BP_OracleSprintReplay", sprintSpeed:900}` → `{"blueprintPath":"/Game/OracleReplay/BP_OracleSprintReplay","sprintSpeed":900,"stateVariable":"bIsSprinting"}` (success; echoes sprintSpeed:900 — looks applied).
3. `blueprint.compile {path:"/Game/OracleReplay/BP_OracleSprintReplay"}` → `{"compiled":true,"status":"UpToDate","errors":[],"warnings":[]}` (clean compile; vars now real on the CDO).
4. `blueprint.get {path:"/Game/OracleReplay/BP_OracleSprintReplay"}` → `defaults:{"bIsSprinting":false,"SprintSpeed":0}` — the `SprintSpeed` variable default is **0**, not the requested 900. (The variable exists with DisplayName "Sprint Speed", category "Sprint".)
5. Control (proves the value DID land elsewhere): `character.get_character_info {blueprintPath:"/Game/OracleReplay/BP_OracleSprintReplay"}` → `{..."customMovementSpeed":900,...}` — `MaxCustomMovementSpeed` is 900, so `sprintSpeed` was applied to the CDO's custom-movement speed but not to the `SprintSpeed` variable it created.

## History
- `#2-triage` `IN-REVIEW` developer — GO (severity Medium unchanged). Verified against synced source: `configure_sprint` (`CharacterHandler.cpp:1244`) creates the `SprintSpeed` float var but writes the speed only to the CDO's `MaxCustomMovementSpeed` (`:1252`) — no `SetBPVarDefaultValue`, and the shared `AddBlueprintVariableChar` helper (`:247`) seeds no default — so the var keeps its float-zero default. Siblings `add_custom_movement_mode` (`:707-708`) and `configure_footstep_fx` (`:879-880`) already persist their value-var defaults; `configure_sprint` alone omits it. Fixing by adding the missing `SetBPVarDefaultValue(Blueprint, TEXT("SprintSpeed"), FString::SanitizeFloat(...))`; adopting the reporter's red test `PinWright.character.configure_sprint.SprintSpeedDefaultPersisted` as the regression gate. Implementation + compile/test verification to follow.
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live against `mcp__pinwright__call` on a fresh `/Game/OracleReplay/BP_OracleSprintReplay` (parent Character). `configure_sprint {sprintSpeed:900}` returned success echoing `sprintSpeed:900`; after `blueprint.compile` (UpToDate, 0 errors) `blueprint.get` read `defaults.SprintSpeed:0` while `get_character_info` read `customMovementSpeed:900` — the passed speed lands on `MaxCustomMovementSpeed` but the created `SprintSpeed` variable keeps its zero default. Source (`CharacterHandler.cpp` L1219-1263) confirms the handler creates `SprintSpeed` at L1244 but never calls `SetBPVarDefaultValue` on it, unlike its sibling `add_custom_movement_mode` (L705-708) which populates its `<Mode>Speed` variable. Not a dup of `B-add-variable-default-value-ignored` (generic `blueprint.add_variable` `defaultValue` param, different method) nor `B-configure-chest-properties-no-cdo-writeback` (those verbs write NOTHING to the CDO; here the CDO write to `MaxCustomMovementSpeed` succeeds and only the created variable's default is dropped). Filed as a silent-wrong-data bug confined to `configure_sprint`.
