---
id: F-vehicle-wheel-asset-batch-properties
title: "No batched property edit for an existing wheel asset — re-tuning N suspension fields forces N single-field set_wheel_asset_property calls"
status: OPEN
severity: Low
category: feature
tags: [vehicle, chaos-vehicle, wheel-asset, suspension, batch, ergonomic]
encounters: 2
lastSeen: 2026-06-24T06:10:13Z
---

# Editing several properties on an already-saved wheel asset has no batch path — one `vehicle.set_wheel_asset_property` call per field

`vehicle.create_wheel_asset` accepts the whole suspension bag in a single
call (`suspensionMaxRaise`, `suspensionMaxDrop`, `springRate`,
`dampingRatio`, plus radius/mass/torque/flags). But once the asset exists,
the only way to write properties back is
`vehicle.set_wheel_asset_property`, whose wiki page is explicit that it
**"Write[s] a single property on a wheel-asset CDO"** — one
`propertyName`/`value` pair per call, each defaulting to `save:true`.

So a *re-tuning* edit — the common iterative-author motion of "the rear
wheels feel too stiff over jumps, bump suspension travel and soften the
spring" — that touches 4 suspension fields on a saved asset costs **4
separate RPC calls**, each its own reflection write + package save round
trip, where one batched call would express the intent. The sibling
`vehicle.set_suspension` *is* exactly this batched suspension verb
(`suspensionMaxRaise`/`suspensionMaxDrop`/`springRate`/`dampingRatio` in
one call), but it only resolves a `UChaosWheeledVehicleMovementComponent`
host (`componentPath`), so it cannot touch a **standalone** wheel asset —
leaving standalone wheel assets with no batched edit at all.

## What it should do

Add a batched setter for an existing wheel asset so the create-time bag
shape is also available for edits, e.g.:

```
vehicle.set_wheel_asset_properties(assetPath,
    properties /* { WheelRadius?, WheelWidth?, SuspensionMaxRaise?,
                    SuspensionMaxDrop?, SpringRate?,
                    SuspensionDampingRatio?, ... } */,
    save? /* one save after the whole batch */)
```

or, more cheaply, accept the same typed suspension/wheel bag that
`vehicle.create_wheel_asset` already takes on a "edit existing asset"
path (apply only the provided fields, single save). Either collapses the
N×(write+save) sequence into one call + one save.

`vehicle.set_wheel_asset_property` (singular) stays as the documented
single-field fast path. This mirrors the established batch precedent
`F-batch-pin-defaults` (DONE — batched `blueprint_graph_set_pin_default_value`).

**Workaround:** Pass all fields at `vehicle.create_wheel_asset` time when
authoring from scratch; for an already-saved asset, issue one
`vehicle.set_wheel_asset_property` per field (the agent did 4: rear
SuspensionMaxRaise, SuspensionMaxDrop, SpringRate, SuspensionDampingRatio).

## History
- `#2-eight-calls-both-wheel-assets` `OPEN` reporter — Additional evidence (same single-field-only standalone wheel-asset edit gap; seed `vehicle.create_wheel_asset`, namespace `vehicle`, outcome `clean`). A 4-wheel off-road-buggy authoring task whose step 5 said "give all four wheels a generous SuspensionMaxRaise of 12 cm and SuspensionMaxDrop of 18 cm, SpringRate around 25000, and a SuspensionDampingRatio of 0.5 … Save the touched wheel assets." Because both standalone wheel assets had to be re-tuned (the `vehicle.set_suspension` host-batch path was unavailable — the vehicle BP did not exist, see `E-vehicle-componentpath-host-discovery` `#5`), the agent issued **8 single-field `vehicle.set_wheel_asset_property` calls** — 4 fields × 2 assets (front: SuspensionMaxRaise=12, SuspensionMaxDrop=18, SpringRate=25000, SuspensionDampingRatio=0.5; rear: same four) — each its own reflection write + package save. One batched `set_wheel_asset_properties(assetPath, {…4 fields…}, save)` per asset would have collapsed this to 2 calls. This doubles the call-count evidence from `#1` (single asset / 4 calls) and confirms the tax scales linearly with the number of standalone wheel assets in a build. Friction is purely the missing batch form — the singular fast-path itself worked cleanly and every field verified via `property.get`. Dedup: ripgrep OPEN+closed — same standalone-wheel-asset batch gap as `#1`; the `componentPath`/host-discovery facet of the same task is tracked in `E-vehicle-componentpath-host-discovery` `#5` (distinct layer); appended here rather than a new file.
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a Chaos rally-car wheel-authoring task (seed `vehicle.create_wheel_asset`, namespace `vehicle`, outcome clean). Distinct PROCESS angle from the judge's discovery/no-steer ticket `E-vehicle-componentpath-host-discovery` (which covers `vehicle.set_suspension` failing on a wheel-asset CDO and steering to `set_wheel_asset_property`): that ticket accepts the single-field fallback as the answer; this one flags that the fallback has *no batched form*, so step 3's single logical edit ("bump suspension travel and soften the spring" → SuspensionMaxRaise=12, SuspensionMaxDrop=16, SpringRate=220, SuspensionDampingRatio=0.45) cost **4 separate `vehicle.set_wheel_asset_property` calls** (each a write+save round trip), where the asymmetry is stark: `vehicle.create_wheel_asset` already accepts all 4 suspension params in one bag, and the host-targeting `vehicle.set_suspension` is itself a 4-field batch — only the standalone-wheel-asset edit path is single-field-only. Friction note (verbatim, the outcome facet judged elsewhere): *"I fell back to vehicle.set_wheel_asset_property for the four suspension fields."* Dedup: ripgrep across OPEN+closed — no `set_wheel_asset_properties`/batch-wheel-property ticket exists; `F-batch-pin-defaults` (DONE) is the analogous batch precedent, not this; `E-vehicle-componentpath-host-discovery` is the discovery/error-steer layer (distinct); `F-vehicle-chaos-typed` (DONE) deliberately shipped the singular fast-path and never specced a batched existing-asset setter. Low severity — modest call-count tax, not a blocker.
