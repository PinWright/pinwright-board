---
id: E-char-var-default-readback-needs-compile
title: "character.md 'Verifying movement settings' sends callers to blueprint.inspect for variable defaults without saying a just-added variable stays invisible there until blueprint.compile"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, character, blueprint, add-custom-movement-mode, variable-default, compile, readback, discovery, var-default-readback-needs-compile]
encounters: 1
lastSeen: 2026-07-11T09:02:17+03:00
rice: [1, 2, 1, 1]
priority: 17
---

# character.md "Verifying movement settings" omits the compile prerequisite for variable-default readback

The character setters that create a Blueprint variable write its value as a durable default on
`NewVariables` via `SetBPVarDefaultValue`
(`Source/PinWright/Private/Handlers/Character/CharacterHandler.cpp:58`) and do not compile:
`add_custom_movement_mode` (`<Mode>Speed`, `CustomModeId_<Mode>`; `:668`, writes at `:706-707`),
`map_surface_to_sound` (`FootstepSoundMap`; `:781`, `:827`), `configure_footstep_fx`
(`FootstepVolumeMultiplier`, `FootstepParticleScale`; `:849`, `:878-879`), and `configure_sprint`'s `SprintSpeed`
write at `:1068`.

`blueprint.get`'s `defaults` and `blueprint.inspect {includeProperties:true}` read the compiled
generated-class CDO, so a variable that is not compiled yet is omitted
(`BuildBlueprintDefaults`, `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp:1362-1397`).
`docs/wiki-src/blueprint.md:245` documents that omission for `blueprint.get`. But
`docs/wiki-src/character.md:13` tells callers to read the footstep variable defaults with
`blueprint.inspect {blueprintPath, includeProperties:true}` and says nothing about compiling
first. A caller who follows it right after the setter sees the value missing and concludes it
did not persist. The reported session spent about 7 extra calls before an unplanned
`blueprint.compile` + re-save made `DashSpeed=1500` visible.

`blueprint.md:245` also names `property.get {includeDefault:true}` as "the authority ... for an
uncompiled variable". That path resolves from class CDOs too
(`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:630-663`), with no
`NewVariables` read, so the claim is unconfirmed; do not route callers to it without a check.

**Workaround:** run `blueprint.compile` after the setter, then read back.

**Fix:** in `docs/wiki-src/character.md` `## Verifying movement settings`, add one note: the
setters that add a variable (`add_custom_movement_mode`, `configure_footstep_fx`,
`map_surface_to_sound`, `configure_sprint`) store a durable default, but `blueprint.inspect` and
`blueprint.get` `defaults` show it only after `blueprint.compile`; link `blueprint.md`'s
`defaults` paragraph. If `property.get {includeDefault:true}` is confirmed to return an
uncompiled variable, name it as the no-compile alternative; otherwise correct `blueprint.md:245`.

**Acceptance:** `character.md` states the compile prerequisite next to the `blueprint.inspect`
route; following the documented steps after `add_custom_movement_mode` on a fresh character
Blueprint shows the `<Mode>Speed` default on the first read-back.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `character.add_custom_movement_mode` task (28 calls, outcome tool_bug — the judge filed the neighbor `B-configure-sprint-speed-var-unset`). This is the distinct PROCESS/docs angle the judge left on the table: after `add_custom_movement_mode` + `asset.save`, the durably-written `DashSpeed=1500` default was absent from both `blueprint.get` `defaults` and `blueprint.inspect {includeProperties:true}` (CDO-sourced readback omits the not-yet-compiled variable), and only surfaced after an unplanned `blueprint.compile` + re-save — a ~7-step recovery detour that would have read as a success-check FAIL without it. The value is durable and not lost (per `SetBPVarDefaultValue` writing `NewVariables.DefaultValue`; judge replay confirmed the setter behaves correctly), so this is "works but non-obvious", filed as a Low docs/discoverability gap. Overlay page to edit = `docs/wiki-src/character.md` (`## Verifying movement settings` section), cross-ref `docs/wiki-src/blueprint.md`. Symptom family `var-default-readback-needs-compile`. Distinct root cause from `E-character-readback-fallback-undocumented` (reader field-omission) though same overlay section, and from `E-blueprint-get-defaults-always-empty` (which now populates `defaults` FROM the CDO and deliberately omits uncompiled vars — this ticket documents that omission for callers).
- `#2-rephrased` `OPEN` developer — Old text cited `Docs/wiki-src/character.md` and CharacterHandler lines that moved, and did not reflect that blueprint.md:245 now documents the uncompiled-variable omission for blueprint.get. Narrowed to the remaining gap: character.md:13 still routes default readback to blueprint.inspect with no compile prerequisite (setters still do not compile, CharacterHandler.cpp:668-727). Added that blueprint.md:245 names property.get {includeDefault} as the uncompiled-variable authority, which source does not support (UtilityPropertyHandler.cpp:630-663 reads only CDOs). Evidence condensed. Severity stays Low. rice 1 1 1 1 -> 1 2 1 1: misleading docs are I=2.
