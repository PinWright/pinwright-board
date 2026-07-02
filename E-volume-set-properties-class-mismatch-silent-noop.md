---
id: E-volume-set-properties-class-mismatch-silent-noop
title: "volume.set_volume_properties returns isError:false + propertiesSet:[] when every requested property is silently dropped because the target volume's class isn't PhysicsVolume/PainCausing/AudioVolume — no error, no warning, no 'skipped' diagnostic"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [volume, set_volume_properties, class-mismatch, propertiesSet, silent-noop, trigger-volume, discoverability]
---

# `volume.set_volume_properties` reports success while silently dropping all class-mismatched properties

`volume.set_volume_properties` applies its documented properties only after a
successful down-cast: `bWaterVolume`/`fluidFriction`/`terminalVelocity`/`priority`
guarded by `Cast<APhysicsVolume>`, `bPainCausing`/`damagePerSec` by
`Cast<APainCausingVolume>`, `bEnabled`/`reverbVolume`/`fadeTime` by
`Cast<AAudioVolume>` (`VolumeHandler.cpp:1364-1387`). When the target volume's
class matches none of those three (the very common `ATriggerVolume` — the class
`volume.create_trigger_volume` produces — but also `ABlockingVolume`,
`APostProcessVolume`, `ALightmassImportanceVolume`, etc.), **every** cast fails,
nothing is added to `PropertiesSet`, and the handler still calls
`Ctx.SendSuccess(Result)` (`:1397`) with `propertiesSet:[]`.

The result is `isError:false` (success) for a call that named valid, documented
properties and applied **none** of them. There is no error, no warning, and no
message saying *which* properties were skipped or *why* (e.g. "TriggerVolume is
not a PhysicsVolume/PainCausingVolume/AudioVolume; these properties do not
apply"). The only signal that the operation was a complete no-op is the empty
`propertiesSet` array — a caller must notice it is empty (vs. the N properties
they requested) to realize nothing happened. A caller who trusts the `isError`
flag concludes the volume was configured when it was not.

This is the request-side companion of `E-volume-get-info-omits-physics-properties`
(the read-side gap: `get_volumes_info` can't echo the physics/pain/audio props).
There, the *write* succeeds but can't be verified via the info verb; here, the
*write* silently does nothing on a class-mismatched volume yet still reports
success. Together they mean a `set_volume_properties` call on a TriggerVolume
both does nothing AND offers no in-band signal of the no-op beyond an empty array.

## Why this is ergonomic (E-), not a tool bug (B-)

Unlike the unconditional fake-success stubs (`B-input-trigger-modifier-stub-silent-success`,
`B-widget-apply-style-silent-noop`, `B-material-stub-handlers-silent-success`),
this handler is **not** a stub and does **not** emit a false affirmative: it
genuinely applies the properties when the class matches, and it reports
`propertiesSet:[]` truthfully — the empty array is not a lie, so an attentive
caller can detect the no-op. The friction is that a class-mismatched request
returns success with no error/warning naming the dropped properties, which is
concretely misleading (success masks a complete no-op) — an ergonomic defect with
a truthful field present, not a fake-success bug.

## What it should do (any of)

- **Warn in-band:** when one or more requested properties could not be applied
  because the volume's class doesn't support them, include a `skipped`/`ignored`
  array (the property names that were dropped) and/or a `note` stating the
  volume's class supports none of them, so the omission is explicit rather than
  inferred from `propertiesSet` being shorter than the request.
- **Fail loud on a total no-op:** if *every* requested property was dropped
  (none applied), return a clean error (e.g. `NOT_APPLICABLE` /
  `CLASS_MISMATCH`: "Trigger_RoomEntry is a TriggerVolume; set_volume_properties
  only configures PhysicsVolume/PainCausingVolume/AudioVolume properties")
  instead of `isError:false` — `propertiesSet:[]` with success on a non-empty
  request is the misleading shape.
- **Docs (`docs/wiki-src/volume.md`):** add a `### volume.set_volume_properties`
  section stating each property family applies only to its volume class, that a
  TriggerVolume (and other non-physics/pain/audio volumes) supports none of them,
  and that the call returns success with `propertiesSet:[]` (a no-op) rather than
  an error on a class mismatch. Same overlay
  `E-volume-get-info-omits-physics-properties`,
  `E-volume-set-extent-units-class-dependent-docs`,
  `E-volume-type-filter-discovery`, `E-volume-create-name-vs-volumename`, and
  `E-volume-get-info-no-limit-spills` already target.

## Verbatim repro (replay-confirmed via `mcp__editor-automation__call`)

Setup — a plain TriggerVolume (the class `create_trigger_volume` makes):
- `volume.create_trigger_volume` `{volumeName:"Trigger_ReplayTest", location:{x:-800,y:600,z:0}, extent:{x:80,y:80,z:120}, rotation:{pitch:0,yaw:45,roll:0}}`
  -> `{... "volumeClass":"ATriggerVolume", "actorClass":"TriggerVolume", "existsAfter":true ...}`

The silent no-op — valid documented props, every one dropped:
- `volume.set_volume_properties` `{volumeName:"Trigger_ReplayTest", bPainCausing:true, damagePerSec:50}`
  -> `{"volumeName":"Trigger_ReplayTest", ..., "actorClass":"TriggerVolume", "propertiesSet":[]}`  (isError:false)
- `volume.set_volume_properties` `{volumeName:"Trigger_ReplayTest", bWaterVolume:true, reverbVolume:0.8, priority:5}`
  -> `{"volumeName":"Trigger_ReplayTest", ..., "actorClass":"TriggerVolume", "propertiesSet":[]}`  (isError:false)

Contrast — schema validation IS enforced for *unknown* names, so the silent path
is specific to valid-but-class-mismatched properties:
- `volume.set_volume_properties` `{volumeName:"Trigger_ReplayTest", totallyMadeUpProperty:123}`
  -> error `[UNKNOWN_PARAMS] Unknown parameter(s) for 'volume.set_volume_properties': [totallyMadeUpProperty]. Valid parameters: [volumeName, bWaterVolume, fluidFriction, terminalVelocity, priority, bPainCausing, damagePerSec, bEnabled, reverbVolume, fadeTime].`

Source-confirmed: the three class-gated property blocks
(`VolumeHandler.cpp:1364-1387`) and the unconditional `Ctx.SendSuccess(Result)`
with `propertiesSet` (`:1389-1397`) — no branch errors or warns when
`PropertiesSet` ends up empty.

## Distinct from

- `E-volume-get-info-omits-physics-properties` (OPEN) — the *read-side* gap
  (`get_volumes_info` never echoes the physics/pain/audio props
  `set_volume_properties` writes). This ticket is the *write-side* class-mismatch
  silent no-op + success on the `set_volume_properties` call itself.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN) — `set_volume_extent`'s
  `extent` units (a different method); that call DOES apply something, the issue
  is the unit meaning. Here nothing is applied at all.
- `B-input-trigger-modifier-stub-silent-success` / `B-widget-apply-style-silent-noop`
  (IN-REVIEW) — unconditional fake-success stubs that emit a false affirmative
  (`triggerSet:true` / "binding created"). This handler is not a stub, applies
  properties correctly when the class matches, and reports `propertiesSet:[]`
  truthfully — hence ergonomic, not a fake-success bug.

## History
- `#1-initial-repro` `OPEN` reporter — Seed-mode tutorial-trigger blockout task (focus `volume.create_trigger_volume`, namespace `volume`, outcome `ergo`; culprit `volume.set_volume_properties`). The task's "tweak the RoomEntry volume's properties for gameplay" step has no fit in `volume.set_volume_properties`: its documented fields are all PhysicsVolume/PainCausing/AudioVolume, none of which apply to a plain TriggerVolume. Replay-confirmed via `mcp__editor-automation__call` on a fresh `Trigger_ReplayTest` TriggerVolume: `set_volume_properties {bPainCausing:true, damagePerSec:50}` and `{bWaterVolume:true, reverbVolume:0.8, priority:5}` both returned `isError:false` with `propertiesSet:[]` (every requested property silently dropped — none applied), while an unknown name (`totallyMadeUpProperty`) correctly returned `[UNKNOWN_PARAMS]`. So valid-but-class-mismatched properties are silently no-op'd with success; only the empty `propertiesSet` array hints at it, with no error/warning naming the skipped props or the class mismatch. Source-confirmed: class-gated casts `VolumeHandler.cpp:1364-1387`, unconditional `SendSuccess` with `propertiesSet` `:1389-1397`. Dedup: ripgrep over the board (`set_volume_properties|class.?mismatch|propertiesSet|PainCausing|silently skip`) — nearest neighbors are `E-volume-get-info-omits-physics-properties` (read-side readback gap), `E-volume-set-extent-units-class-dependent-docs` (different method's units), and the B- fake-success stubs (unconditional false affirmatives) — all distinct (see "Distinct from"). Classified ERGONOMIC (gated, quotable demo above): the call behaves as designed (truthful `propertiesSet:[]`) but the success-on-total-no-op shape is concretely misleading. Fix: in-band `skipped` array / `note`, or fail loud (`NOT_APPLICABLE`/`CLASS_MISMATCH`) when every requested property is dropped, plus a `### volume.set_volume_properties` overlay note.
- `#2-retriage` `OPEN` triage — Low→Medium: success-on-total-no-op masks a misconfig on a class mismatch, though truthful propertiesSet:[] is present; rare path so not High.
- `#3-fix` `IN-REVIEW` developer — Fixed in `Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp` (`volume.set_volume_properties`). Before the three class-gated cast blocks, the handler now classifies each *requested* documented property by its owning class (Physics/Pain/Audio) via `VolumeActor->IsA<...>()` and collects any whose class doesn't match the target volume into a `SkippedProperties` list. After the cast/apply blocks: if `PropertiesSet` is empty but `SkippedProperties` is non-empty (a TOTAL class-mismatch no-op — e.g. physics/pain/audio props on the `ATriggerVolume` `create_trigger_volume` makes), the handler now FAILS LOUD with `Ctx.SendError("CLASS_MISMATCH", ...)` whose error data carries `actorClass` (the volume's real class) and a `skipped` array of the dropped property names, instead of `SendSuccess` with `propertiesSet:[]`. For a PARTIAL mismatch (some props applied, some dropped because they belong to another class), the call still succeeds but the success result now includes a `skipped` array so the omission is explicit rather than inferred from a short `propertiesSet`. Adopts the shipped validate-before-mutate / fail-loud convention used by `ai.configure_slot_behavior` and `gas.set_effect_tags`; the chosen `CLASS_MISMATCH` code matches the ticket's proposed remedy and is distinct from the existing asset-type `ASSET_CLASS_MISMATCH`. Docs: added a `### volume.set_volume_properties` overlay section in `Docs/wiki-src/volume.md` spelling out the per-class property families, the matching create verbs, and the new total/partial/full-match response shapes (surfaces on `call("volume.set_volume_properties")`, costs no tokens on the namespace page). Regression test: `PinWright.volume.set_volume_properties.ClassMismatchFailsLoud` in `Source/PinWright/Private/Tests/World/TestVolumeHandlers.cpp` — spawns a real TriggerVolume via the production `create_trigger_volume` handler, calls `set_volume_properties` with `bWaterVolume`/`priority`/`bPainCausing` against it, and asserts `bSuccess==false`, `ErrorCode=="CLASS_MISMATCH"`, and that the error data's `skipped` array names `bWaterVolume` and `bPainCausing`. If the fail-loud branch is reverted the call returns `isError:false` again and the assertions fire. Did not compile/run (later phase).
- `#4-additional-blocking-volume-priority` `OPEN` reporter — Additional evidence: seed-mode welcome-map dressing task (focus `volume.add_blocking_volume`, namespace `volume`, outcome `ergo`; culprit `volume.set_volume_properties`). Task wanted to bump a freshly-placed `BlockingVolume`'s `priority` to 1 so it takes precedence over overlapping volumes. Replay-confirmed via `mcp__editor-automation__call` the exact `ABlockingVolume` case the ticket named only in passing (line 19), with a positive control proving the same call shape works on a PhysicsVolume: `set_volume_properties {volumeName:"UELogo_BlockingVolume", priority:1}` -> `{"actorClass":"BlockingVolume","propertiesSet":[]}` (isError:false — silent no-op, `priority` dropped); `{volumeName:"UELogo_BlockingVolume", priority:1, bPainCausing:true, damagePerSec:5}` -> `propertiesSet:[]` (all three dropped, still isError:false); but the SAME `{priority:1}` on `DefaultPhysicsVolume0` -> `propertiesSet:["priority"]` (applied). So `priority` (a real, often-needed knob for a blocking barrier's precedence) is silently no-op'd on a BlockingVolume with success, while identical on a PhysicsVolume succeeds visibly — reinforcing the success-on-total-no-op friction with the concrete `priority`-on-BlockingVolume demo. Same disposition (ERGONOMIC, Medium); no severity change.
