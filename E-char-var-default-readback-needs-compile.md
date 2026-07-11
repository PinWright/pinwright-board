---
id: E-char-var-default-readback-needs-compile
title: "character variable-adding setters write a durable variable default, but blueprint.get/inspect readback omits it until blueprint.compile — the 'Verifying movement settings' docs don't warn of the compile prereq"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, character, blueprint, add-custom-movement-mode, variable-default, compile, readback, discovery, var-default-readback-needs-compile]
encounters: 1
lastSeen: 2026-07-11T09:02:17+03:00
---

# character variable-adding setters: the added variable's default is durable but absent from readback until a `blueprint.compile`

The character setters that create a *value* blueprint variable —
`add_custom_movement_mode` (creates `<Mode>Speed`, e.g. `DashSpeed`),
`configure_footstep_fx` (`FootstepVolumeMultiplier`/`FootstepParticleScale`),
`map_surface_to_sound` (`FootstepSoundMap`) — write the passed value as a
**durable** variable default via `SetBPVarDefaultValue` (it lands on the
`FBPVariableDescription.DefaultValue` in `NewVariables`, survives recompiles,
and is serialized by the immediate `asset.save`). That part works and the
per-finding judge replay-confirmed `add_custom_movement_mode` behaves correctly.

The discoverability trap: **`blueprint.get`'s `defaults` map and
`blueprint.inspect {includeProperties:true}`'s properties both source their
values from the compiled generated-class CDO**, and a variable that was just
added but not yet compiled onto the class has no CDO property — so the readback
**omits it entirely** (by design of the `E-blueprint-get-defaults-always-empty`
fix: "a variable not yet compiled onto the class … is omitted rather than
reported wrong"). Result: right after `add_custom_movement_mode` + `asset.save`,
a caller who reads back to confirm the new mode's speed persisted sees the
`DashSpeed` default **absent** from both `blueprint.get` and
`blueprint.inspect` — even though the value is durable on `NewVariables`. The
default only surfaces in readback after an explicit `blueprint.compile` (which
materializes the variable onto the CDO) + re-save.

This is "works but non-obvious", not a defect: the write is durable and the
data is not lost; it is simply invisible to the CDO-sourced readback verbs until
a compile. But the `character` overlay's `## Verifying movement settings`
section already routes callers to `blueprint.inspect {includeProperties:true}`
to read variable defaults (for the footstep-FX scalars) **without mentioning the
compile prerequisite** — so a caller following that documented route on a
freshly-added variable hits the same empty readback and does not know why.

## What it should do (docs only; downstream wiki process)
In `docs/wiki-src/character.md`, extend the `## Verifying movement settings`
section with a compile-prereq note: after a setter that *adds* a blueprint
variable (`add_custom_movement_mode`, `configure_footstep_fx`,
`map_surface_to_sound`), the added variable's default is durable but does **not**
appear in `blueprint.get`'s `defaults` map or `blueprint.inspect
{includeProperties:true}` until the blueprint is compiled (the readback reflects
the compiled CDO; a not-yet-compiled variable is omitted). To confirm a
just-added variable default in-session, run `blueprint.compile` (then re-read).
Cross-reference `docs/wiki-src/blueprint.md` for the general compile-to-CDO
timing. Sibling docs ticket editing the same section:
`E-character-readback-fallback-undocumented` (reader field-omission angle —
distinct root cause, same overlay section).

severity rationale: impact=docs/discoverability (value is durable and not lost; only the CDO-sourced readback omits it until a compile) × reach=character variable-adding setters + a "confirm it persisted" loop (moderate, not every-session) -> Low.

## Evidence (this task — `character.add_custom_movement_mode`, outcome tool_bug; this is the distinct PROCESS/docs angle)
Task: add a `Dash` custom movement mode (customSpeed 1500) plus movement tuning
on `/Game/Global/Blueprints/PlayerCharacter`, then "confirm every one of those
changes actually persisted … so the tuning survives a reload." CallAnalyzer
plan-divergence + call-log ground truth: after `add_custom_movement_mode`
(`customSpeed:1500`, `speedVariable:DashSpeed`, `existsAfter:true`) + `asset.save`
(force=true, 654118 bytes), the `DashSpeed=1500` default was **absent** from
`blueprint.inspect {includeProperties:true}` (a full-text scan of the 134KB
payload found the value 1500 zero times) and from `blueprint.get`'s 17-entry
`defaults` map (`DashSpeed in defaults? False`). It only surfaced after an
**unplanned** `blueprint.compile` (status UpToDate) + a **second** `asset.save`
(654895 bytes): the re-read `blueprint.get` then showed `DashSpeed=1500`,
`CustomModeId_Dash=1` and `blueprint.inspect` showed `DashSpeed={value:1500}`.
The attempt spent ~7 extra steps recovering (searching the inspect payload for
1500; reading the `property.get` / `blueprint.get` / `blueprint.compile` wiki
pages; one extra `blueprint.get`, one `blueprint.compile`, a second `asset.save`,
then re-reading `blueprint.get` + `blueprint.inspect`) purely to make the default
visible — and the task's success check ("customSpeed … persisted as their
default") would have read as FAIL without that compile detour. Source ground
truth: `Plugins/PinWright/Source/PinWright/Private/Handlers/Character/CharacterHandler.cpp`
— `SetBPVarDefaultValue` (L58) writes the durable `DefaultValue` on `NewVariables`;
`add_custom_movement_mode` (L705-708) writes it correctly; the overlay
`Docs/wiki-src/character.md` `## Verifying movement settings` section routes
variable-default reads to `blueprint.inspect {includeProperties:true}` but omits
the compile prerequisite. Reconciled against the judge (which replay-confirmed
`add_custom_movement_mode` works and declined to file it as a bug): filed here
only as the residual pure-discoverability/docs gap, not as a defect.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `character.add_custom_movement_mode` task (28 calls, outcome tool_bug — the judge filed the neighbor `B-configure-sprint-speed-var-unset`). This is the distinct PROCESS/docs angle the judge left on the table: after `add_custom_movement_mode` + `asset.save`, the durably-written `DashSpeed=1500` default was absent from both `blueprint.get` `defaults` and `blueprint.inspect {includeProperties:true}` (CDO-sourced readback omits the not-yet-compiled variable), and only surfaced after an unplanned `blueprint.compile` + re-save — a ~7-step recovery detour that would have read as a success-check FAIL without it. The value is durable and not lost (per `SetBPVarDefaultValue` writing `NewVariables.DefaultValue`; judge replay confirmed the setter behaves correctly), so this is "works but non-obvious", filed as a Low docs/discoverability gap. Overlay page to edit = `docs/wiki-src/character.md` (`## Verifying movement settings` section), cross-ref `docs/wiki-src/blueprint.md`. Symptom family `var-default-readback-needs-compile`. Distinct root cause from `E-character-readback-fallback-undocumented` (reader field-omission) though same overlay section, and from `E-blueprint-get-defaults-always-empty` (which now populates `defaults` FROM the CDO and deliberately omits uncompiled vars — this ticket documents that omission for callers).
