---
id: F-character-info-no-friction-braking
title: "character.get_character_info omits groundFriction and brakingDeceleration — the set_ground_friction / set_braking_deceleration setters can't be read back"
status: IN-REVIEW
severity: Medium
category: feature
tags: [character, movement, get-character-info, ground-friction, braking-deceleration, readback, round-trip]
---

# `character.get_character_info` should report `groundFriction` and `brakingDeceleration`

The `character` namespace exposes dedicated setters for two movement fields that
its lone reader does **not** surface back:

- `character.set_ground_friction` — writes `UCharacterMovementComponent::GroundFriction`
  (`CharacterHandler.cpp:1076`).
- `character.set_braking_deceleration` — writes
  `UCharacterMovementComponent::BrakingDecelerationWalking`
  (`CharacterHandler.cpp:1112`).

`character.get_character_info` (`CharacterHandler.cpp:866`) reads the movement
component and emits `walkSpeed`, `jumpZVelocity`, `airControl`,
`orientToMovement`, `gravityScale`, `customMovementSpeed` (plus capsule, jump
count, camera/spring-arm flags) — but it does **not** emit `groundFriction` or
`brakingDeceleration`, even though both are plain floats on the very same
`UCharacterMovementComponent` it already dereferences (lines 897-906). So a
task that says "set ground friction / braking deceleration, then read the
character info back and confirm the values match" is **impossible to satisfy
through this MCP surface**: the setters succeed and echo their own input, but
the input echo is not an independent readback, and the mandated getter has no
field to confirm against. The two setters are effectively write-only.

This is not a tool bug — `get_character_info` returns valid JSON and is correct
for the fields it does report; `set_ground_friction` and `set_braking_deceleration`
both apply their values correctly. It is a coverage gap in the reader, the same
class of write-only-setter / blind-reader gap already tracked for the networking
namespace in `F-networking-info-no-rpc-detail` (different namespace, different
fields).

## Repro (from this task's call log)
1. `blueprint.create` `{name:"BP_PlayerHero", savePath:"/Game/Heroes", parentClass:"Character"}` -> ok.
2. `character.set_ground_friction` `{BP_PlayerHero, groundFriction:12.5}` -> ok (echoes `groundFriction:12.5`).
3. `character.set_braking_deceleration` `{BP_PlayerHero, brakingDeceleration:2600}` -> ok.
4. `character.set_walk_speed:750`, `set_jump_height:600`, `set_gravity_scale:1.15` -> all ok.
5. `character.get_character_info` `{BP_PlayerHero}` -> returns `walkSpeed:750`,
   `jumpZVelocity:600`, `gravityScale:1.1499999` (==1.15), capsule/camera fields,
   but **no** `groundFriction` and **no** `brakingDeceleration` field at all.

Three of the five seeded movement values round-trip; `groundFriction` (12.5) and
`brakingDeceleration` (2600) cannot be confirmed through the getter the task
mandates.

## Proposed extension (in-place, additive)
Add two number fields to the `get_character_info` movement block, right beside
the existing `walkSpeed`/`gravityScale` lines (`CharacterHandler.cpp:900-905`),
preserving all existing fields:

```cpp
Result->SetNumberField(TEXT("groundFriction"), Movement->GroundFriction);
Result->SetNumberField(TEXT("brakingDeceleration"), Movement->BrakingDecelerationWalking);
```

(Field name `brakingDeceleration` chosen to mirror the `set_braking_deceleration`
param; reads `BrakingDecelerationWalking`, the field that setter writes.)

**Acceptance check:** after the repro above, `get_character_info` returns
`groundFriction:12.5` and `brakingDeceleration:2600`.

**Workaround until implemented:** none in-band — re-running the setter only
re-asserts intent (echoes the passed value, not stored state), and there is no
sibling character reader. `python.execute` against the CDO's movement component
would read the live floats, but that is a source-level fallback, not the getter
the task names.

## History
- `#1-initial-audit` `OPEN` reporter — Process/coverage friction surfaced by a third-person-character setup task (agent_fail; judge filed nothing). Discovery was smooth (wiki index + dedicated `set_*`/`get_character_info` pages gave exact methods first read; all 5 setters succeeded with zero retries) — the friction is purely the verification gap: `get_character_info` (`CharacterHandler.cpp:866-942`) emits walkSpeed/jumpZVelocity/airControl/orientToMovement/gravityScale/customMovementSpeed + capsule/jump/camera fields but OMITS `groundFriction` and `brakingDeceleration`, so the seeded `set_ground_friction(12.5)` and `set_braking_deceleration(2600)` values cannot be round-trip-confirmed through the mandated getter. Both fields are plain floats on the same `UCharacterMovementComponent` already dereferenced (GroundFriction written at `:1076`, BrakingDecelerationWalking at `:1112`), so the fix is two additive `SetNumberField` lines. Same write-only-setter / blind-reader class as `F-networking-info-no-rpc-detail` but a distinct namespace/method/fields, so filed separately. Not a tool bug — getter is correct for what it reports, setters apply correctly; filed as a reader coverage feature gap.
- `#2-additional-moon-platformer` `OPEN` reporter — Reproduced independently from a low-gravity "moon platformer" task (seed `character.set_gravity_scale`; finding lands on neighbor `character.get_character_info`). Created `/Game/MoonPlatformer/BP_MoonWalker` (Character), applied all 5 setters (gravityScale 0.25, walkSpeed 350, jumpZVelocity 1200, groundFriction 3.0, brakingDeceleration 1500), then replayed `character.get_character_info {blueprintPath:"/Game/MoonPlatformer/BP_MoonWalker"}` -> confirms the omission verbatim: `{"walkSpeed":350,"jumpZVelocity":1200,"airControl":0.05,"orientToMovement":false,"gravityScale":0.25,"customMovementSpeed":600,"maxJumpCount":1,...}` — NO `groundFriction`, NO `brakingDeceleration` field. 3 of 5 values round-trip; the friction-3.0 and braking-1500 writes are unconfirmable through the mandated getter. Note correcting this ticket's "Workaround: none in-band": the attempt agent DID confirm both omitted values in-band without `python.execute`, via `blueprint.inspect {includeProperties:true}` reading the CharacterMovement CDO (showed GroundFriction=3, BrakingDecelerationWalking=1500) — so `blueprint.inspect` is a working (if non-obvious / cross-namespace / verbose) round-trip fallback, but the dedicated character reader the task names still cannot confirm 2 of 5 fields.
- `#3-additional-footstep-polish` `OPEN` reporter — Reproduced a third time, now against a **stock Epic content character** (not a freshly-created BP) from a footstep-polish task (seed `character.configure_footstep_fx`; finding again lands on neighbor `character.get_character_info`). Replayed `character.get_character_info {blueprintPath:"/Game/ExampleContent/Animation_Basics/BP_RootMotionCharacter.BP_RootMotionCharacter"}` after a `configure_movement_speeds {walkSpeed:450, groundFriction:8.0}` write -> `{"blueprintPath":".../BP_RootMotionCharacter","assetName":"BP_RootMotionCharacter","capsuleRadius":30,"capsuleHalfHeight":90,"walkSpeed":450,"jumpZVelocity":420,"airControl":0.05,"orientToMovement":false,"gravityScale":1,"customMovementSpeed":600,"maxJumpCount":1,"useControllerRotationYaw":true,"hasSpringArm":false,"hasCamera":false}` — `walkSpeed:450` round-trips but there is **NO `groundFriction`** field (and no `brakingDeceleration`), so the `groundFriction:8.0` write is unconfirmable through the mandated getter, exactly as this ticket describes. Related secondary gap from the same task: `configure_footstep_fx {volumeMultiplier:1.5, particleScale:2}` now persists its scalars (the `SetBPVarDefaultValue` stub fix from `B-map-surface-to-sound-no-effect`, IN-REVIEW, is present in source at `CharacterHandler.cpp:849-853`), but `get_character_info` also surfaces none of the footstep-FX values (`FootstepVolumeMultiplier`/`FootstepParticleScale`) — its `movementVariables` filter only matches names containing Speed/Movement or bIs/bCan prefixes, so the Footstep* vars never appear. Same write-only/blind-reader class; if this fix also extends `get_character_info` to a footstep-FX block it would close that companion gap. Not a tool bug — the getter returns valid JSON and is correct for what it reports.
- `#4-fix` `IN-REVIEW` developer — Implemented the additive two-field fix exactly as proposed. Added `Result->SetNumberField(TEXT("groundFriction"), Movement->GroundFriction)` and `Result->SetNumberField(TEXT("brakingDeceleration"), Movement->BrakingDecelerationWalking)` to the `get_character_info` movement block in `Source/EditorAutomationRpcGateway/Private/Handlers/Character/CharacterHandler.cpp` (right after the existing `gravityScale`/`customMovementSpeed` lines, on the same already-dereferenced `UCharacterMovementComponent`). Reads `BrakingDecelerationWalking` — the exact field `set_braking_deceleration` writes (`:1112`); `groundFriction` mirrors `set_ground_friction`'s write (`:1076`). Purely additive — all existing getter fields preserved, no caller breakage. Scope held to the core two fields; the footstep-FX companion gap (history #3) is left to the `E-`/footstep tracks. Regression test: added `FCharacterGetCharacterInfoFrictionBrakingRoundTripTest` (`EditorAutomationRpcGateway.character.get_character_info.FrictionBrakingRoundTrip`) in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestCharacterHandlers.cpp` — creates an in-memory `ACharacter` blueprint, drives the real `set_ground_friction {12.5}` / `set_braking_deceleration {2600}` setters, then invokes `get_character_info` with response capture and asserts the result carries `groundFriction==12.5` and `brakingDeceleration==2600`; it fails if the two new getter lines are reverted (fields absent). Did not compile/run (later phase). Note for the verifier: the ticket body's "Workaround: none in-band" is a known wording error — `blueprint.inspect {includeProperties:true}` is a working cross-namespace readback (per history #2 / `E-character-readback-fallback-undocumented`); this fix closes the gap through the mandated character getter regardless.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
