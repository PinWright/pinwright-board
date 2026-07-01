---
id: F-vehicle-chaos-typed
title: "Typed Chaos vehicle authoring (wheel asset CRUD, suspension, engine, transmission)"
status: DONE
severity: Medium
category: feature
tags: [physics, chaos-vehicle, wheeled-vehicle, authoring, suspension, transmission]
---

# Typed Chaos vehicle authoring RPCs

`physics.configure_vehicle` is a single opaque RPC that accepts a
free-form `wheels` array, a `vehicleType` string ("car" / "bike" /
"tank" / "aircraft"), and `engine` / `transmission` blobs, then
dispatches `CreateVehicle ...` editor console commands. It does not
touch real `UChaosWheeledVehicleMovementComponent` /
`UChaosVehicleWheel` state and offers no iterative tuning loop.

Authoring a real `AWheeledVehiclePawn` (Chaos vehicles plugin) needs
three things the current RPC cannot express:

1. **Wheel asset CRUD** — `UChaosVehicleWheel` is a `UObject` asset
   that owns radius, mass, friction multiplier, lateral/longitudinal
   slip stiffness, max steer angle, max brake/handbrake torque,
   suspension max raise/drop, spring rate, damping ratio, sweep
   shape, and ABS/TC flags. No way to create, duplicate, or edit
   one today.

2. **Per-wheel setup on the movement component** —
   `FChaosWheelSetup { WheelClass, BoneName, AdditionalOffset,
   bDisableSteering }`. Bone binding and wheel class swap is what
   distinguishes a Civic from a monster truck on the same mesh.

3. **Engine / transmission / steering curves on the movement
   component** — `FVehicleEngineConfig { MaxRPM, MaxTorque,
   TorqueCurve, EngineBrakeEffect, EngineRevUpMOI, EngineRevDownRate
   }`, `FVehicleTransmissionConfig { bUseAutomaticGears,
   ForwardGearRatios, ReverseGearRatios, FinalRatio, ChangeUpRPM,
   ChangeDownRPM, GearChangeTime }`, and `FVehicleSteeringConfig {
   SteeringType, AngleRatio, SteeringCurve }`. All are
   `FRuntimeFloatCurve`-bearing structs that the generic
   `set_property` path cannot edit.

Without these, the only thing `configure_vehicle` can produce is a
stub actor with no drivable handling — useless as a starting point
for tuning.

## Proposed scope

Mirror the depth of `skeleton.create_physics_asset` so the
iterative tuning loop (edit → preview → measure → edit) is fully
RPC-driven. Keep `physics.configure_vehicle` as the orchestrator;
add typed primitives underneath it.

```
vehicle.create_wheel_asset(path, parentClass?, radius?, width?, mass?,
    frictionMultiplier?, lateralSlipStiffness?, longitudinalSlipStiffness?,
    maxSteerAngle?, maxBrakeTorque?, maxHandbrakeTorque?,
    suspensionMaxRaise?, suspensionMaxDrop?, springRate?, springPreload?,
    dampingRatio?, sweepShape?, sweepType?, bAffectedByBrake?,
    bAffectedByHandbrake?, bAffectedBySteering?, bAffectedByEngine?,
    bABSEnabled?, bTractionControlEnabled?, save?)
  -> { assetPath, className }

vehicle.set_wheel_asset_property(assetPath, propertyName, value, save?)
  # fast-path for single-field tuning sweeps without re-passing full struct

vehicle.set_wheel_setup(componentPath, wheelIndex, wheelClass?, boneName?,
    additionalOffset?, bDisableSteering?, compile?, save?)
  # operates on UChaosWheeledVehicleMovementComponent::WheelSetups[wheelIndex];
  # auto-grows the array if wheelIndex == Num()

vehicle.remove_wheel_setup(componentPath, wheelIndex, compile?, save?)

vehicle.set_engine_curve(componentPath, maxRPM?, maxTorque?,
    torqueCurve? /* [{rpm, torqueNormalized}, ...] */,
    engineBrakeEffect?, engineRevUpMOI?, engineRevDownRate?, save?)

vehicle.set_transmission(componentPath, bUseAutomaticGears?,
    forwardGearRatios? /* [float] */, reverseGearRatios? /* [float] */,
    finalRatio?, changeUpRPM?, changeDownRPM?, gearChangeTime?,
    transmissionEfficiency?, save?)

vehicle.set_steering_config(componentPath, steeringType?, angleRatio?,
    steeringCurve? /* [{speedKph, steerScale}, ...] */, save?)

vehicle.set_suspension(componentPath, wheelIndex?, /* all-wheels if omitted */
    suspensionMaxRaise?, suspensionMaxDrop?, springRate?, springPreload?,
    dampingRatio?, suspensionForceOffset?, save?)
```

`componentPath` accepts both Blueprint subobject paths (e.g.
`/Game/Vehicles/BP_Car.BP_Car_C:VehicleMovement`) and live actor
component paths, matching existing `component.*` RPCs.

`physics.configure_vehicle` stays as the high-level convenience
wrapper but reimplements its body on top of the typed primitives —
deletes the `CreateVehicle` console-command path entirely.

## Why one ticket, not three

Wheel-asset CRUD, suspension, and engine/transmission tuning all
share:

- Same `ChaosVehicles` / `ChaosVehiclesEditor` module dependency
  (already conditionally added via `TryAddConditionalModule` in
  `Build.cs` if present, otherwise needs to be added).
- Same `FRuntimeFloatCurve` editing helper (needed by torque, brake,
  steering curves — write once).
- Same `UChaosWheeledVehicleMovementComponent::ComputeConstants()`
  rebuild + `Modify()` + dirty-package handshake.
- Same iterative tuning loop — a vehicle author edits a wheel, runs
  PIE, tweaks engine, runs PIE. Splitting forces three round-trips
  across tickets when one developer would naturally implement them
  together.

Compare `skeleton.create_physics_asset` + `physics.add_body` +
`physics.add_constraint` — shipped as one feature family in
`PhysicsHandler.cpp`.

## Fix

Implement in a new `Handlers/Physics/ChaosVehicleHandler.cpp`
gated on `__has_include("ChaosVehicleMovementComponent.h")` so the
plugin still builds on projects without the ChaosVehicles plugin
enabled. Add a `ChaosVehiclesEditor` / `ChaosVehicles` entry to
`TryAddConditionalModule()` in `Build.cs`. Reuse
`PropertyUtils::SetRuntimeFloatCurve` (extract from existing curve
edit helpers if not already extracted; otherwise add).

## History
- `#1-initial-spec` `OPEN` reporter — Audit of `physics.configure_vehicle` showed it dispatches `CreateVehicle` console commands rather than configuring real `UChaosWheeledVehicleMovementComponent` / `UChaosVehicleWheel` state, with no path to edit suspension, engine torque curves, transmission gear ratios, or steering. Three coupled gaps (wheel asset CRUD, per-wheel setup, engine/transmission/steering/suspension tuning) filed as one ticket to match the cohesive iterative-tuning loop of `skeleton.create_physics_asset`. Broad UE5 audience: relevant to anyone authoring drivable vehicles via Chaos.
- `#2-implemented-narrowed-scope` `IN-REVIEW` developer — Implemented narrowed scope: `vehicle.create_wheel_asset`, `vehicle.set_wheel_asset_property`, `vehicle.set_wheel_setup`, `vehicle.remove_wheel_setup`, `vehicle.set_suspension` in `Handlers/Physics/ChaosVehicleHandler.cpp` (gated by `__has_include("ChaosVehicleMovementComponent.h")`). Added new `Utils/ComponentPathUtils.{h,cpp}` to resolve BP-CDO subobject paths and live actor component paths. Extended `Build.cs` `TryAddConditionalModule` to also probe `Plugins/Experimental/<XxxPlugin>/Source/<Y>/` (ChaosVehicles lives under `ChaosVehiclesPlugin/Source/`); added ChaosVehicles + ChaosVehiclesEditor entries. Engine/transmission/steering curves and `physics.configure_vehicle` rewrite deferred to a sibling ticket F-vehicle-chaos-curves (TODO file separately). Tests in `Tests/Gameplay/TestChaosVehicleHandlers.cpp` cover wheel-asset round-trip + bad-parent-class counterfactual.
- `#3-verify-typed-vehicle-rpcs` `DONE` tester — Verified: all 5 RPC wiki pages (`vehicle.create_wheel_asset`, `vehicle.set_wheel_asset_property`, `vehicle.set_wheel_setup`, `vehicle.remove_wheel_setup`, `vehicle.set_suspension`) returned full param schemas. End-to-end: `vehicle.create_wheel_asset path=/Game/App/UI/Test/W_McpVerifyTemp_F-vehicle-chaos-typed radius=35 mass=20 maxBrakeTorque=1500` returned `success:true, parentClass:ChaosVehicleWheel, existsAfter:true`; `vehicle.set_wheel_asset_property propertyName=WheelRadius value=42.5` returned `success:true`. Cleaned up via `asset.delete`.
