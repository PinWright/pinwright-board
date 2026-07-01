---
id: E-volume-get-info-omits-physics-properties
title: "volume.get_volumes_info echoes only name/class/location/extent — it never reads back the physics/pain/audio properties set by volume.set_volume_properties, so a PhysicsVolume's bWaterVolume/fluidFriction/terminalVelocity/priority config can't be verified via the info call"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, get_volumes_info, set_volume_properties, readback, projection, physics-volume, water, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `volume.get_volumes_info` has no readback for the properties `volume.set_volume_properties` writes

`volume.set_volume_properties` can configure a whole family of volume-class
properties — `bWaterVolume`, `fluidFriction`, `terminalVelocity`, `priority`
(PhysicsVolume), `bPainCausing`/`damagePerSec` (PainCausingVolume),
`bEnabled`/`reverbVolume`/`fadeTime` (AudioVolume) — see its registration
(`VolumeHandler.cpp:1335-1347`). But `volume.get_volumes_info`, the natural
"read back the volumes and confirm their config" verb, emits **only four fields
per row**: `name`, `class`, `location`, `extent` (`VolumeHandler.cpp:1531-1547`
for the `AVolume` loop, `:1570-1586` for the `ATriggerBase` loop). None of the
`set_volume_properties` property families is ever serialized into the readback.

The consequence: after configuring a physics/water volume, the **only** evidence
that the config took effect is the `set_volume_properties` write echo itself —
there is no independent info-call readback to verify it. A caller who wants to
confirm "is this volume actually set to water with friction 0.3 / terminal
velocity 1000 / priority 1?" via `get_volumes_info` cannot: those fields simply
aren't in the payload. This is the same readback-projection gap family as
`E-audio-get-info-soundclass-mix-readback-thin`,
`E-niagara-inspect-no-param-readback-projection`, and
`E-get-ai-info-no-perception-readback` — a write verb whose effects the
corresponding info verb cannot surface.

## What's wrong

`volume.get_volumes_info` (`VolumeHandler.cpp:1486`) builds each volume row from
`GetActorLabel()`, `GetClass()->GetName()`, `GetActorLocation()`, and
`GetActorBounds()` only (`:1531-1547`, `:1570-1586`). It never down-casts to
`APhysicsVolume` / `APainCausingVolume` / `AAudioVolume` to read the very
properties `set_volume_properties` is documented to set on those classes. So the
write side and the read side are asymmetric: you can set `bWaterVolume=true`,
`fluidFriction=0.3`, `terminalVelocity=1000`, `priority=1`, but the canonical
info verb shows you nothing but the box's location and extent.

There IS a generic escape hatch (`actor.get_component_property` /
`actor.describe` on the volume's root component or the actor), but a caller
working in the `volume.*` namespace has no signal that the namespace's own info
verb is property-blind — they reasonably expect "get_volumes_info confirms the
volumes I just configured."

## What it should do

- **Method (preferred):** when a row's class is a `set_volume_properties`-aware
  type, include the corresponding property block in the row — e.g. a
  `physicsProperties: {bWaterVolume, fluidFriction, terminalVelocity, priority}`
  object on `APhysicsVolume` rows (and analogous `painProperties` /
  `audioProperties` blocks for the other two families), or gate it behind an
  opt-in `fields`/`includeProperties` flag so the default payload stays small
  (note `E-volume-get-info-no-limit-spills` already wants a `fields`/`namesOnly`
  projection on this same verb — the two projections compose: `namesOnly` for
  cheap readback, `includeProperties` for the verify-my-physics-config case).
- **Docs (`docs/wiki-src/volume.md`):** until/unless the method echoes them, add a
  `### volume.get_volumes_info` note that the info call returns only
  name/class/location/extent and does **not** echo the properties set by
  `volume.set_volume_properties`; to verify `bWaterVolume`/`fluidFriction`/etc.,
  read the `set_volume_properties` response or use `actor.get_component_property`
  / `actor.describe` on the volume. This is the same overlay
  `E-volume-get-info-no-limit-spills`, `E-volume-type-filter-discovery`,
  `E-volume-create-name-vs-volumename`, and
  `E-volume-set-extent-units-class-dependent-docs` already target — a
  `### volume.get_volumes_info` section serves several of these wants at once.

## Evidence

From the hazard-layer arena blockout struggle audit (focus
`volume.create_kill_z_volume`, namespace `volume`, 9 calls, outcome clean). The
task created a `Moat_Water` PhysicsVolume and configured it via
`volume.set_volume_properties {bWaterVolume:true, fluidFriction:0.3,
terminalVelocity:1000, priority:1}`, then the story explicitly asked the final
`volume.get_volumes_info` readback to "confirm all three new volumes … with the
bounds/extents we set." Friction note, verbatim: *"get_volumes_info reports only
location/extent and does NOT echo physics properties
(bWaterVolume/fluidFriction/terminalVelocity/priority), so the moat's water
config could only be confirmed from the set_volume_properties response, not the
readback - a discoverability gap if a user wants to verify physics props via the
info call."* No call errored — pure verification/discoverability overhead: the
agent could confirm extents from the readback but had to fall back to the
write-echo for the physics config because the info verb is property-blind.
Source-confirmed: `set_volume_properties` writes those fields
(`VolumeHandler.cpp:1335-1347`) but `get_volumes_info` rows carry only
name/class/location/extent (`:1531-1547`, `:1570-1586`).

## Distinct from

- `E-volume-get-info-no-limit-spills` (OPEN) — that is the *size/Read-tax* on the
  same verb (no `limit`/projection so the full-level dump spills past 10000
  chars); this is the *field coverage* gap (a specific property family is never
  in the payload at all). They share the verb and the `fields` projection idea
  but are different defects: even a perfectly-sized, single-row readback would
  still omit the physics properties.
- `B-blocking-volume-no-brush-geometry` (IN-REVIEW) — that is the brush/extent
  ground-truth bug; this is purely the *physics-property readback* coverage,
  independent of whether the extents are correct.
- `E-volume-set-extent-units-class-dependent-docs` (OPEN) — that is the
  request-side units meaning of `set_volume_extent`'s `extent`; this is the
  read-side coverage of `set_volume_properties`'s outputs.
- `F-water-actor-authoring` (DONE) — that adds a `water.*` namespace for rendered
  WaterBody/WaterZone actors; this is about reading back the `bWaterVolume`
  property on a stock `APhysicsVolume`, a different surface entirely.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the hazard-layer arena blockout task (focus `volume.create_kill_z_volume`, namespace `volume`, 9 calls, outcome clean). Distinct PROCESS angle (the judge filed nothing for this clean task): `volume.set_volume_properties` writes PhysicsVolume `bWaterVolume`/`fluidFriction`/`terminalVelocity`/`priority` (and PainCausing/Audio families) per `VolumeHandler.cpp:1335-1347`, but `volume.get_volumes_info` rows carry only name/class/location/extent (`:1531-1547` AVolume loop, `:1570-1586` ATriggerBase loop) and never down-cast to read those properties back — so the task's `Moat_Water` water config could be confirmed only from the `set_volume_properties` write echo, not the unfiltered final readback the story asked for. Friction note verbatim: *"get_volumes_info reports only location/extent and does NOT echo physics properties (bWaterVolume/fluidFriction/terminalVelocity/priority), so the moat's water config could only be confirmed from the set_volume_properties response, not the readback - a discoverability gap if a user wants to verify physics props via the info call."* Proposed: add a class-aware `physicsProperties`/`painProperties`/`audioProperties` block to `get_volumes_info` rows (or an opt-in `includeProperties`/`fields` flag composing with the `namesOnly` projection `E-volume-get-info-no-limit-spills` already wants), plus a `### volume.get_volumes_info` note in `docs/wiki-src/volume.md` that the info call omits `set_volume_properties` outputs and pointing to `actor.get_component_property`/`actor.describe` as the current workaround. Same readback-projection family as `E-audio-get-info-soundclass-mix-readback-thin`/`E-niagara-inspect-no-param-readback-projection`/`E-get-ai-info-no-perception-readback`. Distinct from `E-volume-get-info-no-limit-spills` (size/Read-tax on the same verb), `B-blocking-volume-no-brush-geometry` (brush ground-truth), `E-volume-set-extent-units-class-dependent-docs` (request-side units), and `F-water-actor-authoring` (DONE, rendered water actors).
</content>
