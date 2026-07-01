---
id: E-character-walk-run-speed-alias
title: "configure_movement_speeds exposes walkSpeed and runSpeed as distinct params but both write MaxWalkSpeed; passing both silently drops one"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [character, movement, param-alias, walk-speed, run-speed, silent-overwrite]
---

# `character.configure_movement_speeds` presents `walkSpeed` and `runSpeed` as two speeds but they are aliases for one field

`character.configure_movement_speeds` declares two separate, differently-described
numeric parameters: `walkSpeed` ("Max walk speed") and `runSpeed` ("Max run speed
(sets MaxWalkSpeed)"). A caller naturally reads these as two independent speeds —
a walk speed and a (faster) run/jog speed — which is the standard character-movement
distinction. They are not independent: both write the single
`UCharacterMovementComponent::MaxWalkSpeed` field, so they are aliases. When both
are passed in one call, the handler applies `walkSpeed` then unconditionally
overwrites it with `runSpeed` (`CharacterHandler.cpp:374-377`):

```cpp
if (RawPayload->HasField(TEXT("walkSpeed")))
    Movement->MaxWalkSpeed = static_cast<float>(Ctx.GetNumber(TEXT("walkSpeed"), 600.0));
if (RawPayload->HasField(TEXT("runSpeed")))
    Movement->MaxWalkSpeed = static_cast<float>(Ctx.GetNumber(TEXT("runSpeed"), 600.0));   // clobbers walkSpeed
```

So `runSpeed` always wins when both are supplied, and `walkSpeed` is silently
discarded. There is no warning, no error, and no echo of the resulting speed in the
result payload (the response is the generic
`{...,"existsAfter":true,"assetClass":"Blueprint"}` with no speed fields), so the
caller has no in-band signal that one of their two values was thrown away. The
mislead is compounded by `get_character_info`, whose `walkSpeed` field reports
`MaxWalkSpeed` — i.e. it reports whichever of the two "won", labeled as `walkSpeed`,
not as the param the user actually set.

This is purely an ergonomic / API-shape issue, not a tool bug: the call returns
valid JSON, modifies the blueprint, and the `runSpeed→MaxWalkSpeed` mapping is
documented. But exposing two distinct, distinctly-described params that collide on
one field with no diagnostic when both are passed is concretely misleading — a
caller who sets `walkSpeed:250, runSpeed:650` ends up with a single `MaxWalkSpeed`
of 650 while believing they configured both a 250 walk and a 650 run.

## What it should do
Either (a) collapse to a single documented param (keep `runSpeed` or rename the
shared slot to `maxWalkSpeed`, and mark `walkSpeed` an explicit alias of it in the
docs), or (b) if both names are kept, reject the ambiguous case — return
`INVALID_PARAMS` when both `walkSpeed` and `runSpeed` are present (they target the
same field) — and/or echo the final applied speed(s) in the result so the
overwrite is visible. At minimum, the param descriptions should state that
`walkSpeed` and `runSpeed` both set `MaxWalkSpeed` and that `runSpeed` takes
precedence, so the collision is discoverable from the wiki page.

## Verbatim repro
1. `blueprint.create` `{name:"BP_AgileScout", savePath:"/Game/Characters", parentClass:"Character"}` → ok.
2. `character.configure_movement_speeds` `{blueprintPath:"/Game/Characters/BP_AgileScout", walkSpeed:250, runSpeed:650, crouchSpeed:220, acceleration:2400, groundFriction:8}`
   → `{"blueprintPath":"/Game/Characters/BP_AgileScout","assetPath":"/Game/Characters/BP_AgileScout","assetName":"BP_AgileScout","existsAfter":true,"assetClass":"Blueprint"}` (no speed echoed).
3. `character.get_character_info` `{blueprintPath:"/Game/Characters/BP_AgileScout"}`
   → `{..."walkSpeed":650,...}` — the `walkSpeed` field reports 650 (the `runSpeed` value); the `walkSpeed:250` that was passed is gone.
4. Control: `character.configure_movement_speeds` `{...,"walkSpeed":250}` (runSpeed omitted) then `get_character_info`
   → `{..."walkSpeed":250,...}` — confirms `walkSpeed` alone writes `MaxWalkSpeed`, i.e. the two params target the same field and the step-2 `runSpeed` simply overwrote the `walkSpeed`.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on `/Game/Characters/BP_AgileScout` (parent Character). `configure_movement_speeds {walkSpeed:250, runSpeed:650, ...}` returned success with no speed echo; `get_character_info` then reported `walkSpeed:650` (the runSpeed value), and a control call with `walkSpeed:250` alone reported `walkSpeed:250` — proving `walkSpeed` and `runSpeed` are both aliases for `MaxWalkSpeed` (`CharacterHandler.cpp:374-377`, runSpeed applied last so it wins) and that passing both silently drops the `walkSpeed` value with no diagnostic. Param descriptions present them as two distinct speeds ("Max walk speed" / "Max run speed"), which is the misleading framing. Not a tool bug (valid JSON, blueprint modified, runSpeed→MaxWalkSpeed is documented); filed as ergonomic param-alias collision.
- `#2-additional-guard-npc` `OPEN` reporter — Independently re-confirmed from a patrolling-guard-NPC task (seed `character.configure_nav_movement`; finding lands on neighbor `character.configure_movement_speeds`). Created `/Game/AI/BP_GuardNPC` (parent Character), then replayed `configure_movement_speeds {walkSpeed:250, runSpeed:600, crouchSpeed:150, acceleration:1500, groundFriction:8}` → success with no speed echo (`{"blueprintPath":"/Game/AI/BP_GuardNPC","assetPath":"/Game/AI/BP_GuardNPC","assetName":"BP_GuardNPC","existsAfter":true,"assetClass":"Blueprint"}`); `get_character_info` then reported `"walkSpeed":600` — the `runSpeed` value, so the passed `walkSpeed:250` is gone. Control: `configure_movement_speeds {walkSpeed:250}` alone → `get_character_info` reported `"walkSpeed":250`, confirming both params target the same `MaxWalkSpeed` and `runSpeed` (applied second) clobbers `walkSpeed`. Source line refs have shifted since #1: the two colliding writes are now `CharacterHandler.cpp:503-506` and the getter reads `Movement->MaxWalkSpeed` into the `walkSpeed` field at `CharacterHandler.cpp:900`. A guard-NPC author wanting a calm 250 patrol walk plus a 600 chase run ends up with a single 600 walk speed and no in-band signal the 250 was dropped. Confirms the existing repro and disposition; no new file.
- `#3-fix` `IN-REVIEW` developer — Fixed via the board's canonical validate-before-mutate + echo pattern (the same shape shipped for B-data-table-row-values-silent-drop / E-water-underwater-settings-silent-drop). In `configure_movement_speeds` (`CharacterHandler.cpp`): (a) before any mutation, reject the ambiguous both-present case — if BOTH `walkSpeed` and `runSpeed` are passed, `SendError("INVALID_PARAMS", ...)` explaining they are aliases of the single `MaxWalkSpeed` and that `configure_sprint` is the real second speed (no silent drop); (b) on the single-speed success path, echo the applied ground speed in the result as `walkSpeed` (== resulting `MaxWalkSpeed`), matching the sibling `set_walk_speed`/`configure_sprint` setters which already echo their value (the result previously echoed no speed); (c) tightened both param descriptions to state they both set `MaxWalkSpeed`, are aliases (pass one), and point to `configure_sprint`. Wiki: added one note to the existing `## Verifying movement settings` section of `docs/wiki-src/character.md` documenting the alias, the both-present rejection, the echo, and `configure_sprint`. Regression test `EditorAutomationRpcGateway.character.configure_movement_speeds.WalkRunSpeedAliasRejectsBothAndEchoes` (in `TestCharacterHandlers.cpp`) drives the real handler against an in-memory ACharacter blueprint: asserts both-present → `INVALID_PARAMS` (not a silent success), single `walkSpeed:250` → success echoing `walkSpeed:250`, and single `runSpeed:650` → success echoing `walkSpeed:650` (proving runSpeed→MaxWalkSpeed) — it fails if either the guard or the echo is reverted. The pre-existing `configure_movement_speeds.ValidParamsNoCrash` smoke test was updated to pass a single speed (it previously passed both). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Character/CharacterHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestCharacterHandlers.cpp`, `docs/wiki-src/character.md`.
