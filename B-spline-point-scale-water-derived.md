---
id: B-spline-point-scale-water-derived
title: "spline.set_spline_point_scale on a water spline writes a value the engine re-derives — width survives read-back and save, then reverts on load"
status: IN-REVIEW
severity: High
category: bug
tags: [spline, water, derived-state, persistence, misleading-success, silent-revert]
---

# A water spline's point scale is derived; writing it is a doomed write

A river-width change was applied via `spline.set_spline_point_scale`, read back at the
expected 3490.6, saved, and committed — then reverted to 4800 on the next level load.

This is the worst persistence shape: the write **lands**, **reads back** at the requested
number, and **saves** into the map package. Only a genuine reload can see the loss, and
`editor.open_level` on the already-open map does not reload.

## Root cause (source-verified against `C:\UE_5.8`)

UE Water stores width and depth in `UWaterSplineMetadata` (`RiverWidth` / `Depth`) and derives
the spline point scale from them.
`UWaterSplineComponent::SynchronizeWaterProperties` (`WaterSplineComponent.cpp:189`) assigns
`Scale.X = RiverWidth` (`:231-236`) and `Scale.Y = Depth` (`:238-244`) and writes the result
back with `SetScaleAtSplinePoint` (`:246`). `PostLoad` calls it (`:26-39`) — and so do
`PostDuplicate` (`:41-53`), `PostEditChangeProperty` (`:152-163`) and `PostEditImport`
(`:165-172`), so `actor.duplicate` loses the value too, not only a reload.

The engine states the fact itself: `UWaterSplineComponent` overrides
`AllowsSplinePointScaleEditing() const { return false; }` (`WaterSplineComponent.h:54`)
against the base `USplineComponent`'s `return true` (`SplineComponent.h:425`).

The authoritative setters are `UWaterBodyRiverComponent::SetRiverWidthAtSplineInputKey` /
`SetRiverDepthAtSplineInputKey` (`WaterBodyRiverComponent.h:51,54`, `WATER_API`), which had no
PinWright verb.

## Fix

- `spline.set_spline_point_scale` refuses with a new registered `DERIVED_PROPERTY` code when
  `!SplineComp->AllowsSplinePointScaleEditing()`. The gate is the **engine's own predicate**,
  not a hand-maintained type list, so a spline type that starts deriving its scale in a later
  engine version is covered with no code change. Water is identified by reflection
  (`FindObject<UClass>(nullptr, TEXT("/Script/Water.WaterSplineComponent"))`) so the `spline.*`
  namespace keeps its zero Water dependency. The refusal is total — nothing is written — and
  the error payload carries `derivedWrite {property, derivedFrom, authoritativeVerb,
  survivesReload:false, explanation}`. On the success path the response now echoes the scale
  read back off the component instead of the request.
- New `water.set_river_width_at_spline_point` and `water.set_river_depth_at_spline_point`:
  write the metadata through the engine setters, then
  `UWaterSplineComponent::K2_SynchronizeAndBroadcastDataChange` (`WaterSplineComponent.h:62`)
  so the engine re-derives the scale and `UWaterBodyComponent::OnWaterSplineDataChanged`
  (`WaterBodyComponent.cpp:1382`, bound at `:1453`) rebuilds the body. Each point reports the
  `stored` metadata value **and** the `derivedScale` the engine computed; `applied` is their
  conjunction with the request, and a point that disagrees returns `APPLY_FAILED` with the
  evidence rather than success. `derivedScale` is written by the engine's Synchronize, not by
  the handler, so it is a check the write path cannot fake.
  Selector is exactly one of `pointIndex` or `allPoints:true` — neither has a safe default
  (point 0 would silently edit one point of a river meant to be widened end to end; all points
  would flatten a taper meant to be kept). `width`/`depth` must be > 0, because the engine
  floors the stored value at `KINDA_SMALL_NUMBER`.

Refused rather than forwarded: forwarding a `Scale.X` write is exact only for rivers —
`RiverWidth` is editable on rivers alone (`WaterSplineMetadata.cpp:21-24`) while `Scale.X` is
synchronized from it on every water body type, so a forwarded lake write would store a value
that controls nothing and report it as a width change.

## Acceptance

`spline.set_spline_point_scale` on a river fails with `DERIVED_PROPERTY` naming
`water.set_river_width_at_spline_point` and leaves `Scale.X` untouched; the authoritative verb
sets a width that is still there after a genuine level reload.

## Related

- `E-water-spline-default-points-undocumented` — same component, documentation gap.
- `B-compile-material-landscape-consumers-stale` — sibling "success reported, effect absent",
  one layer up (durable edit that reaches no consumer). Both are recorded in
  `Docs/rpc-design.md` §5a / §5b.

## History
- `#1-initial-repro` `OPEN` reporter — River width set via `spline.set_spline_point_scale`, read back at the expected 3490.6, saved and committed; reverted to 4800 on the next level load. UE Water stores width in spline metadata and re-derives `Scale.X` from it, so writing `Scale.X` is a write to a derived value — invisible to read-back, invisible to a save check, only a genuine reload catches it.
- `#2-fix` `IN-REVIEW` developer — Verified the mechanism in engine source: `UWaterSplineComponent::SynchronizeWaterProperties` (`WaterSplineComponent.cpp:189`) assigns `Scale.X` from `UWaterSplineMetadata::RiverWidth` (`:231-236`) and `Scale.Y` from `Depth` (`:238-244`), called by `PostLoad` (`:26-39`), `PostDuplicate` (`:41-53`), `PostEditChangeProperty` (`:152-163`) and `PostEditImport` (`:165-172`) — so `actor.duplicate` lost the value too, which the original report did not know. `spline.set_spline_point_scale` now refuses with the newly registered `DERIVED_PROPERTY` code, gated on the engine's own `USplineComponent::AllowsSplinePointScaleEditing()` (`SplineComponent.h:425`, overridden to false at `WaterSplineComponent.h:54`) rather than on a type list, and carries a structured `derivedWrite` block naming the remedy. Chose refuse over forward: `RiverWidth` is editable on rivers only (`WaterSplineMetadata.cpp:21-24`) while `Scale.X` is synchronized on every water body type, so forwarding a lake write would store a value that controls nothing. Shipped the missing remedy in the same change — `water.set_river_width_at_spline_point` / `water.set_river_depth_at_spline_point` write `UWaterBodyRiverComponent::SetRiverWidthAtSplineInputKey` / `SetRiverDepthAtSplineInputKey` (`WaterBodyRiverComponent.h:51,54`) then `K2_SynchronizeAndBroadcastDataChange` (`WaterSplineComponent.h:62`) so the scale is re-derived and the body rebuilds, and report per point both the `stored` metadata value and the engine-written `derivedScale`. Test: `Private/Tests/World/TestWaterSplineScaleIsDerived.cpp` asserts the scale write FAILS with `DERIVED_PROPERTY`, that the payload names the authoritative verb, that `Scale.X` is unchanged by the refused call, and that the authoritative verb moves `Scale.X` measured off the component. It carries no Water headers — the fixture is built through the production verbs — so it compiles where the plugin is disabled and skips at runtime on `WATER_PLUGIN_NOT_AVAILABLE`. Docs: `wiki-src/water.md`, `wiki-src/spline.md` and `wiki-src/level-building.terrain-and-water.md` document derived-versus-authoritative; `Docs/rpc-design.md` §5a records the transferable rule. Not compiled or run — later integration phase.
