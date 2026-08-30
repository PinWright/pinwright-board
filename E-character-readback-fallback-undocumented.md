---
id: E-character-readback-fallback-undocumented
title: "character wiki gives no signal that get_character_info can't confirm footstep-FX or nav-agent fields — callers waste navs hunting the blueprint.inspect/property.get fallback"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, character, movement, readback, round-trip, get-character-info, blueprint-inspect, property-get, footstep, nav-agent, discovery]
---

# `character` wiki gives no signal that `get_character_info` can't confirm the footstep-FX or nav-agent fields, nor names the working fallback

The `character` namespace's lone reader is `get_character_info`, but several
`character` setters write fields it does **not** surface. A caller who follows the
natural "apply all setters, then read the character info back and confirm each
value" loop finds the reader silently short — with **no docs signal** that this
will happen and **no pointer** to the fallback that does work.

Specifically, `get_character_info` (`CharacterHandler.cpp:889-944`) does NOT
report:
- The **footstep-FX scalars** written by `character.configure_footstep_fx`:
  `FootstepVolumeMultiplier` / `FootstepParticleScale` (set as blueprint
  variables at `CharacterHandler.cpp:846-853`). The reader's only variable output
  is the `movementVariables` filter (`:931-944`), which matches names with a
  `bIs`/`bCan` prefix or containing `Speed`/`Movement` — `Footstep*` matches none,
  so the values never appear.
- The **nav-agent fields** written by `character.configure_nav_movement`:
  `NavAgentProps.AgentRadius` / `NavAgentProps.AgentHeight` and `bUseRVOAvoidance`
  (set at `CharacterHandler.cpp:736-741`). `get_character_info` reads none of them.

(Note: `groundFriction`/`brakingDeceleration` USED to be on this list, but the
reader now emits both directly — `CharacterHandler.cpp:910-911`, landed via
`F-character-info-no-friction-braking #4-fix`. So they round-trip through
`get_character_info` and are no longer part of this gap.)

The overlay (`docs/wiki-src/character.md`) is currently a 4-line namespace blurb:
it documents none of the individual `set_*`/`configure_*` methods, says nothing
about which fields `get_character_info` returns vs omits, and names no
verification fallback. So the caller discovers the gap the expensive way — and
then has to discover the *fix* the expensive way too. There are working in-band
round-trips (`blueprint.inspect {includeProperties:true}` on the blueprint reads
the footstep-FX variable defaults; `property.get` / `blueprint.inspect
{includeProperties:true}` on the CharacterMovement CDO surfaces
`NavAgentProps.AgentRadius`/`AgentHeight` and `bUseRVOAvoidance`), but they are
cross-namespace, non-obvious, and undocumented for this purpose, so the caller
cannot reach for them directly.

The wiki note is a cheap, durable mitigation: it sets the expectation up front
(which fields `get_character_info` confirms vs omits) and routes a caller who must
verify the footstep-FX / nav-agent values straight to the working fallback instead
of cycling through unrelated read verbs.

## Evidence (this task)
Two cross-task struggle-audits anchor the remaining gap (the original
ground-friction/braking detour from `#1` is now superseded — that field set
round-trips through `get_character_info` since `F-#4-fix`):

- **Footstep-FX** (`character.configure_footstep_fx`, namespace `character`,
  outcome gap): task demanded "apply footstep FX (volumeMultiplier 1.5 /
  particleScale 2), map the Default surface to a sound, set walkSpeed 450 /
  groundFriction 8.0, then read the character info back so I can confirm the
  footstep-FX and movement values stuck." Discovery and write phase were smooth
  (all mutation calls succeeded, zero retries). `get_character_info` round-tripped
  only `walkSpeed:450` and surfaced NONE of `volumeMultiplier` (1.5) or
  `particleScale` (2.0) — unconfirmable through the mandated reader, with no docs
  signal and no fallback pointer.
- **Nav-agent** (`character.configure_nav_movement`, namespace `character`,
  outcome ergo): patrolling-guard-NPC task created `/Game/AI/BP_GuardNPC`, ran six
  `configure_*` setters (nav radius 45 / height 190 / RVO true, capsule, speeds,
  crouch, orient), then "read the blueprint back with `character.get_character_info`
  and report the resulting nav agent radius/height, RVO avoidance flag…".
  `get_character_info` round-tripped capsule/walkSpeed/orientToMovement but OMITTED
  the three nav fields the task explicitly demanded. The agent fell back to two
  read-only `property.get` calls — `CharacterMovement.NavAgentProps`
  (AgentRadius=45, AgentHeight=190) and `CharacterMovement.bUseRVOAvoidance`
  (=true) — to confirm the round-trip; no docs signal up front, no pointer to the
  fallback.

## Fix (docs only; downstream wiki process)
In `docs/wiki-src/character.md`, add a short "verifying movement settings" note:
- `get_character_info` is the `character` reader, but it surfaces only
  `walkSpeed` (MaxWalkSpeed), `jumpZVelocity`, `airControl`, `orientToMovement`,
  `gravityScale`, `customMovementSpeed`, `groundFriction`, `brakingDeceleration`
  (plus capsule/jump/camera flags) — it does **not** report the
  `configure_footstep_fx` scalars (`FootstepVolumeMultiplier` /
  `FootstepParticleScale`) or the `configure_nav_movement` fields
  (`NavAgentProps.AgentRadius` / `AgentHeight`, `bUseRVOAvoidance`), so those
  cannot be confirmed through it.
- To round-trip the footstep-FX scalars, read the blueprint's variable defaults
  with `blueprint.inspect {blueprintPath, includeProperties:true}` (fields
  `FootstepVolumeMultiplier` / `FootstepParticleScale`).
- To round-trip the nav-agent fields, read the CharacterMovement CDO with
  `property.get` / `blueprint.inspect {includeProperties:true}` on
  `CharacterMovement` (fields `NavAgentProps.AgentRadius` / `AgentHeight`,
  `bUseRVOAvoidance`).

Distinct PROCESS angle from the tool/feature tickets that add fields to the
reader; this is the docs/discovery layer that tells callers the verification limit
up front and names the working fallback. Same shape as
`E-sequencer-set-track-state-no-readback-doc` (the docs companion to
`F-sequencer-track-state-readback`) and
`E-blueprint-get-omits-components-readback-guidance` (readback-routing docs).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of task `character.set_gravity_scale` (moon-platformer BP_MoonWalker, 20 calls, outcome gap). The judge filed the tool/feature side (`F-character-info-no-friction-braking`, reader omits 2 fields); this is the distinct PROCESS/docs angle: `docs/wiki-src/character.md` (3-sentence blurb) gives no signal that `get_character_info` cannot confirm `set_ground_friction`/`set_braking_deceleration` and names no fallback, so the agent paid a 5-call verification detour — 4 extra wiki-navs (`configure_movement_speeds`, `blueprint.get`, `blueprint.scs.get`, `blueprint.inspect`) plus one corrective `blueprint.inspect {includeProperties:true}` read of the CharacterMovement CDO — after a write phase that had zero retries. Proposed a one-line overlay note stating the `get_character_info` field coverage and routing ground-friction/braking confirmation to `blueprint.inspect {includeProperties:true}`; cross-links the `F-` feature. Overlay page to edit = `docs/wiki-src/character.md`. Docs-only ergonomic gap.
- `#2-extends-to-footstep-fx` `OPEN` reporter — Cross-task evidence that this docs gap is wider than ground-friction/braking: it also covers the `configure_footstep_fx` scalars. Struggle-audit of task `character.configure_footstep_fx` (namespace `character`, 12 calls, outcome gap) against the stock Epic BP `/Game/ExampleContent/Animation_Basics/BP_RootMotionCharacter`. The task explicitly demanded "apply footstep FX (volumeMultiplier 1.5 / particleScale 2), map the Default surface to a sound, set walkSpeed 450 / groundFriction 8.0, then read the character info back so I can confirm the footstep-FX and movement values stuck." Discovery and the write phase were smooth (wiki gave all params first read; all 4 mutation calls succeeded with zero retries — only one self-correcting typo `asset.find`→suggestion-list→`asset.search`). The friction was again purely verification: `character.get_character_info` round-tripped only `walkSpeed:450` and surfaced NONE of `groundFriction` (8.0), `volumeMultiplier` (1.5), or `particleScale` (2.0) — so 3 of the 4 just-written values are unconfirmable through the mandated reader, with no docs signal up front and no pointer to a fallback. Verbatim friction note: "get_character_info does not surface groundFriction or the footstep-FX values, so only walkSpeed=450 was directly confirmable via readback (the other writes are confirmed only by their non-error responses)." The one-line overlay note this ticket proposes should therefore also list the footstep-FX scalars (`FootstepVolumeMultiplier`/`FootstepParticleScale`) among the fields `get_character_info` does NOT report, and route their confirmation to `blueprint.inspect {includeProperties:true}` (CDO `FootstepVolumeMultiplier`/`FootstepParticleScale`) the same way it routes ground-friction/braking. Tool/feature side already aggregated on `F-character-info-no-friction-braking #3-additional-footstep-polish`; this is the durable docs/discovery angle. Overlay page to edit = `docs/wiki-src/character.md`.
- `#3-extends-to-nav-agent-rvo` `OPEN` reporter — Cross-task evidence the gap also covers the `configure_nav_movement` fields (navAgentRadius / navAgentHeight / RVO avoidance), a THIRD field set `get_character_info` can't confirm. Struggle-audit of task `character.configure_nav_movement` (namespace `character`, outcome ergo; judge filed the walk/run-speed alias side on `E-character-walk-run-speed-alias #2`). Patrolling-guard-NPC task: create `/Game/AI/BP_GuardNPC` (Character), run six configure_* setters (nav radius 45 / height 190 / RVO true, capsule 45/95, speeds, crouch, orient-to-movement), then "read the blueprint back with character.get_character_info and report the resulting nav agent radius/height, RVO avoidance flag, capsule dimensions, walk/run speeds." Discovery + write phase were smooth — every configure call succeeded first-try, zero retries — but `get_character_info` round-tripped capsule (45/95), walkSpeed, orientToMovement, friction and OMITTED the three nav fields the task explicitly demanded (navAgentRadius / navAgentHeight / avoidanceEnabled never appear in its output). The agent fell back to two cross-namespace read-only `property.get` calls — `CharacterMovement.NavAgentProps` (AgentRadius=45, AgentHeight=190) and `CharacterMovement.bUseRVOAvoidance` (=true) — to confirm the round-trip; no docs signal up front and no pointer to the fallback. Verbatim friction note: "character.get_character_info omits the nav fields the task asked to confirm (navAgentRadius/navAgentHeight/avoidanceEnabled never appear in its output), so I had to fall back to read-only property.get on CharacterMovement.NavAgentProps/bUseRVOAvoidance to confirm the round-trip." Note this fallback is a different verb than histories #1/#2 (`property.get` on the CDO movement component, vs `blueprint.inspect {includeProperties:true}`) — both work, neither is named in the overlay. The proposed `docs/wiki-src/character.md` note should therefore ALSO list the nav fields (`NavAgentProps.AgentRadius`/`AgentHeight`, `bUseRVOAvoidance`) among what `get_character_info` does NOT report, and name `property.get`/`blueprint.inspect` on `CharacterMovement` as the confirmation route. Same durable docs/discovery angle; overlay page to edit = `docs/wiki-src/character.md`.
- `#4-reword-and-fix` `IN-REVIEW` developer — Re-scoped the ticket: the `#1` ground-friction/braking premise is now STALE — `get_character_info` emits both fields directly (`CharacterHandler.cpp:910-911`, landed via `F-character-info-no-friction-braking #4-fix`), so they round-trip and are no longer part of this gap. Re-anchored the title/body/**Fix** onto the field sets `get_character_info` GENUINELY still omits: the footstep-FX scalars `FootstepVolumeMultiplier`/`FootstepParticleScale` (`configure_footstep_fx` writes them at `CharacterHandler.cpp:846-853`; the reader's `movementVariables` filter at `:931-944` matches only `bIs`/`bCan`/`Speed`/`Movement` names, so `Footstep*` never surfaces) and the nav-agent fields `NavAgentProps.AgentRadius`/`AgentHeight`/`bUseRVOAvoidance` (`configure_nav_movement` writes them at `:736-741`; the reader reads none). Implemented the docs fix: added a `## Verifying movement settings` section to `Docs/wiki-src/character.md` listing what `get_character_info` confirms (incl. groundFriction/brakingDeceleration now) vs omits, routing the footstep-FX scalars to `blueprint.inspect {includeProperties:true}` (blueprint variable defaults) and the nav-agent fields to `property.get` / `blueprint.inspect` on `CharacterMovement`. Regression test `Tests/Infra/TestCharacterReadbackFallbackDocs.cpp` (`FCharacterReadbackFallbackNamespaceDocTest`) renders the live `character` namespace page through `WikiHandler::RenderPage` (the path the gateway uses) and asserts the section + the footstep-FX/nav markers + fallbacks survive; it fails if the overlay note is reverted. Files: `Docs/wiki-src/character.md`, `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestCharacterReadbackFallbackDocs.cpp`. Docs-only; no handler code changed.
- `#5-extends-to-rotation-rate` `OPEN` reporter — Cross-task evidence the gap also covers the `character.configure_rotation` `rotationRate` (yaw) field — a FOURTH field set `get_character_info` can't confirm (plus the spring-arm `TargetArmLength` set via `configure_camera_component`). Struggle-audit of a third-person-feel task (seed `character.configure_rotation`; finding lands on neighbor `character.get_character_info`) against the stock Epic BP `/Game/ExampleContent/Input_Examples/Blueprints/BP_Pixel_Dude_Character`. The task explicitly mandated the read-back: "read the character info back so I can confirm the rotation block actually took: orient-to-movement true, use-controller-yaw false, and rotation rate yaw 540." Discovery + write phase were smooth (all configure_* calls succeeded, zero retries). Replayed `character.get_character_info {blueprintPath:"/Game/ExampleContent/Input_Examples/Blueprints/BP_Pixel_Dude_Character"}` after the writes -> `{"capsuleRadius":15,"capsuleHalfHeight":25,"walkSpeed":600,"jumpZVelocity":600,"airControl":0.2,"orientToMovement":true,"gravityScale":2,"customMovementSpeed":600,"groundFriction":8,"brakingDeceleration":2000,"navAgentRadius":15,"navAgentHeight":50,"avoidanceEnabled":false,"maxJumpCount":1,"useControllerRotationYaw":false,"hasSpringArm":true,"hasCamera":true}` — `orientToMovement:true` and `useControllerRotationYaw:false` round-trip, but there is **NO `rotationRate` field at all** (and no spring-arm `TargetArmLength`, no pitch/roll flags), so the headline `rotationRate:540` write — the value the task most explicitly asked to confirm — is unconfirmable through the mandated reader. The agent fell back to the cross-namespace read-only `property.get {objectPath:".../BP_Pixel_Dude_Character.Default__BP_Pixel_Dude_Character_C", propertyName:"CharacterMovement.RotationRate"}` -> `value:[0,540,0]` (replay-confirmed verbatim) to verify the round-trip; no docs signal up front, no pointer to the fallback. (Note: a first `property.get` on the `..._C` BlueprintGeneratedClass path failed `[PROPERTY_NOT_FOUND]` — `CharacterMovement` lives on the `Default__..._C` CDO, not the generated class; the working fallback is the CDO path.) The `## Verifying movement settings` overlay note this ticket adds should therefore ALSO list the rotation-rate field (`CharacterMovement.RotationRate`, written by `configure_rotation`) and the spring-arm `TargetArmLength` (written by `configure_camera_component`) among what `get_character_info` does NOT report, routing their confirmation to `property.get` on the `Default__..._C` CDO. Same durable docs/discovery angle; tool/feature side belongs on `F-character-info-no-friction-braking` (reader could additively emit `rotationRate`). Verbatim friction note: "the prescribed reader character.get_character_info does NOT surface RotationRate/rotationRate yaw (nor spring arm TargetArmLength), so the rotation-rate==540 round-trip could not be confirmed through the namespace's own reader; I had to fall back to property.get on the blueprint CDO to verify it." Overlay page to edit = `docs/wiki-src/character.md`.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
