---
id: B-derived-state-report-contract-gaps
title: "Small contract gaps in the derived-state verbs: empty authoritativeVerb, width verb named for a depth write, remedy/notRefreshed contradiction, missing point_index alias, fixed float tolerance"
status: OPEN
severity: Low
rice: [1, 1, 1, 1]
priority: 8
category: bug
tags: [water, spline, material-authoring, derived-state, wire-contract, conventions]
---

# Five small gaps in the derived-state verbs' wire contract

None of these produce a false success. They are places where the structured payload is less useful
than the prose beside it, or where a convention is missed. Paths are relative to
`Source/PinWright/Private/`.

## 1. A refusal can name no verb at all

`Handlers/Geometry/SplineHandler.cpp:154` resets `OutAuthoritativeVerb` and only the
`UWaterSplineComponent` branch sets it (`:176`). `docs/rpc-design.md` §5a says a refusal must name a
verb that exists, and its "ask the engine, do not maintain a type list" rule (`:172`) is what makes
the empty branch reachable: any other spline type whose `AllowsSplinePointScaleEditing()` returns
false gets `authoritativeVerb: ""`.

**Fix:** when no verb is known, put the generic route in the field (`property.set` on the
authoritative component, or the discovery page) instead of an empty string.

## 2. `authoritativeVerb` names the width verb for a `Scale.Y` (depth) write

`SplineHandler.cpp:176` always returns `water.set_river_width_at_spline_point`, though Scale.Y is
depth. `docs/wiki-src/spline.md:72-73` presents `authoritativeVerb` as the replacement verb, so a depth write is
sent to the width verb. Only the `explanation` prose (`:177-184`) names the depth verb.

**Fix:** choose the verb from the scale component the caller changed (Y →
`water.set_river_depth_at_spline_point`).

## 3. `remedy` / `notRefreshed[]` contradict the header's contract

`Utils/DerivedStateReport.h:97-99` documents `Remedy` as "What a caller should do about anything in
NotRefreshed. Empty when the coverage is complete." `AddKnownUnrefreshedConsumers`
(`Handlers/Material/MaterialLandscapeConsumers.h:326-337`, called from
`MaterialAuthoringHandler.cpp:3654` and `:3781`) always adds the "open asset editors" entry to
`NotRefreshed`, but sets `Remedy` only when `!IsComplete()`, and that remedy names
`landscape.set_material`, which does nothing for asset editors. A clean run therefore emits
`complete: true`, a non-empty `notRefreshed[]` and no `remedy`. The entry is also added when
`bMeasured` is false.

**Fix:** split "out of scope by design" from "in scope and missed" into two fields, or reword the
header contract to match the behaviour.

## 4. `pointIndex` has no `point_index` alias

`Handlers/Water/WaterHandler.cpp:686` reads `GetIntFirstOf({ TEXT("pointIndex") })` and the
registrations (`:810`, `:833`) declare no alias, so the dispatcher refuses `point_index` with
`UNKNOWN_PARAMS` (`Dispatch/RpcDispatcher.cpp:171-196`). The refusal is clear, but the plugin
`CLAUDE.md` convention says handlers accept both camelCase and snake_case.

**Fix:** declare `point_index` as an alias on both registrations and add it to the
`GetIntFirstOf` list.

## 5. Fixed `0.01` apply tolerance against `float` storage

`WaterHandler.cpp:725` stores `static_cast<float>(Value)` and `:735` compares the read-back with a
fixed `0.01` tolerance. Above about 65,000 the float ULP is larger than 0.01, so a correct write
would report `APPLY_FAILED`. Real river widths are far below that, so this is a latent edge.

**Fix:** use a relative tolerance (or one float ULP of the value).

## 6. Citation drift

- `WaterSplineComponent.h:54` should be `:53` (`AllowsSplinePointScaleEditing`) in
  `docs/rpc-design.md:172` and `SplineHandler.cpp:139`.
- `WaterSplineComponent.h:62` should be `:60` (`K2_SynchronizeAndBroadcastDataChange`) in
  `WaterHandler.cpp:522`.
- `MaterialLandscapeConsumers.h:34` cites `LandscapeEdit.cpp:710` as
  `ULandscapeComponent::GetCombinationMaterial`; `:710` is a call site, the definition is at `:583`.
- The emitter comment at `DerivedStateReport.h:139-140` omits `consumersWithNothingToRefresh`, which
  the function emits (`:158`).

**Acceptance:** a scale write on a non-water spline with scale editing disabled returns a non-empty
`authoritativeVerb`. A Scale.Y write on a water spline returns
`water.set_river_depth_at_spline_point`. A clean `compile_material` run returns a `consumerRefresh`
block that is consistent with the documented `Remedy` contract. `point_index` is accepted. A width of
100000 applies without `APPLY_FAILED`. The citations above match the engine source.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the derived-state-honesty changeset
  (`099b83b3`..`0fe35187`) against `Docs/rpc-design.md` during integration pass 11. Items 4 and the
  `SafePoint` question were re-verified directly by the integration pass; items 1-3, 5 and the
  citation drift are as reported by the audit and are **not** independently re-verified.
- `#2-rephrased` `OPEN` developer — Item 3's code moved from `MaterialAuthoringHandler.cpp` to `AddKnownUnrefreshedConsumers` (`Handlers/Material/MaterialLandscapeConsumers.h:326-337`, called at `MaterialAuthoringHandler.cpp:3654`, `:3781`), so the ticket no longer spans two files. Item 4's harm was wrong: `point_index` is no longer silently dropped into a misleading XOR error, because the dispatcher refuses it with `UNKNOWN_PARAMS` (`RpcDispatcher.cpp:171-196`). Only the convention gap remains. Verified items 1, 2 and 5 at `SplineHandler.cpp:154`, `:176` and `WaterHandler.cpp:725`, `:735`. Re-verified the citation drift against UE 5.8 (`WaterSplineComponent.h:53`, `:60`; `GetCombinationMaterial` defined at `LandscapeEdit.cpp:583`). Refreshed all citations, added per-item Fix and an Acceptance line. Severity unchanged (Low).
