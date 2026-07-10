---
id: E-lighting-spawn-light-stationary-setter-noop
title: "lighting.spawn_light spawns a Stationary light; movable-gated dynamic setters silently no-op with no mobility param or wiki warning"
status: OPEN
severity: Medium
category: ergonomic
tags: [movable-gated-setter-noop, docs, lighting, spawn_light, mobility]
encounters: 1
lastSeen: 2026-07-11T02:22:48+0300
---

# `lighting.spawn_light` produces a Stationary light whose dynamic setters silently no-op

`lighting.spawn_light` creates its light actor at the engine-default **Stationary**
mobility (a spawned `ASpotLight` sets `SpotLightComponent->Mobility =
EComponentMobility::Stationary`). But the common "dial in the look" setters an
agent then calls on the `LightComponent` — `SetAttenuationRadius`,
`SetInnerConeAngle`, `SetOuterConeAngle` (and the cast-shadow toggle) — are gated
inside the engine by `AreDynamicDataChangesAllowed()`, which **rejects a
non-Movable light**. On a freshly-spawned Stationary light those setters do
nothing yet each call still returns success. The wiki page for `lighting.spawn_light`
lists only `lightClass`/`lightType`/`name`/`location`/`rotation`/`properties` —
**no `mobility` param and no warning** that the dynamic setters require Movable.

This is a discoverability / soft-blocker gap, not a broken tool: once mobility is
flipped to Movable the setters apply and round-trip correctly (the audited task
verified all six values on readback). The friction is that discovering the
requirement forced a **UE engine C++ source dive** — reading `SpotLight.cpp`
(mobility default) and `LightComponent.cpp` (the `AreDynamicDataChangesAllowed()`
guard) — plus inserting an unplanned `property.set` `Mobility=Movable` step (and a
Mobility readback) before the setters would take. A less-careful agent skipping
that step would have produced a spotlight where the throw radius and both cone
angles never applied while every RPC reported success — exactly the silent
false-success this iteration's success check hunts for.

**Compounding factor (`object.call_function` void:true masking):** the six setters
were invoked via `object.call_function`, which returns only `{function, void:true}`
for a void mutator UFunction. `void:true` signals only that the reflected function
executed, **not** that its effect took — an engine-internal guard no-op (the
Stationary rejection) is indistinguishable in the response from a real apply. So
the mobility gate is invisible at the call site; the caller only learns it by
reading the property back.

## What it should do

- **Ergonomic (preferred):** `lighting.spawn_light` should accept a `mobility`
  param (and/or default freshly-spawned dynamic lights whose look is meant to be
  set to `Movable`), so the everyday "spawn a spot and dial it in" gesture doesn't
  silently drop three settings.
- **Docs (minimum):** in `docs/wiki-src/lighting.md`, on the `lighting.spawn_light`
  entry, warn that spawned lights default to Stationary and that
  attenuation-radius / inner+outer cone / cast-shadow setters require Movable
  mobility or they silently no-op — steer callers to set `Mobility=Movable` first.
  Add a companion caveat on the `docs/wiki-src/object.md` `object.call_function`
  page: `void:true` means "the function executed", not "state changed" — always
  read a mutating setter's property back to confirm it took, especially on
  components with mobility/edit-condition guards.

**Workaround:** set the `LightComponent` `Mobility=Movable` via `property.set`
before calling the dynamic setters, then read each property back to confirm.

severity rationale: impact=soft-blocker (doable only via an engine source dive to
avoid a silent false-success) × reach=not-every-session (lighting authoring is a
common but not per-session path) -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — STRUGGLE (process) audit of the "hero-prop
  spotlight" task (outcome nonrepro; the Judge disproved a separate cold-load
  "persistence loss" misdiagnosis and filed nothing on the OUTCOME axis). Distinct
  PROCESS angle, kept as pure discoverability per the audit rules: `lighting.spawn_light`
  spawns a Stationary spot light and the movable-gated dynamic setters
  (`SetAttenuationRadius`/`SetInnerConeAngle`/`SetOuterConeAngle`) silently no-op on
  it while returning `void:true`, with no `mobility` param or wiki warning. The
  agent avoided the trap by source-diving `SpotLight.cpp` + `LightComponent.cpp`
  and inserting an unplanned `property.set Mobility=Movable` (RESULT applied:true)
  before the setters, then six `actor.get_component_property` readbacks confirmed
  all values persisted. No wasted `is_error` call — pure discoverability / source-dive
  overhead. `object.call_function` `void:true` masking is the mechanism that makes
  the gate invisible at the call site. Dedup (ripgrep over OPEN+closed; qmd
  unavailable): distinct from `E-level-spawn-light-no-sky-type` (different method
  `level.spawn_light`, Sky-enum gap, not mobility) and from `F-object-call-function`
  (DONE — the call primitive itself; not the mobility/void-true discoverability).
  No existing `movable-gated-setter-noop` family sibling on the board. Docs target:
  `docs/wiki-src/lighting.md` (+ a `object.md` void:true caveat).
