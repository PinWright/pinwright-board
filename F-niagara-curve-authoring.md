---
id: F-niagara-curve-authoring
title: "Typed Niagara curve / distribution helpers"
status: DONE
severity: Medium
category: feature
tags: [niagara, curve, data-interface, authoring]
---

# Typed Niagara curve / distribution helpers

Curve-driven effects (trail fade, color-over-life, force-curve falloff,
size-by-velocity) are pervasive in Niagara authoring. They are backed by
two distinct data shapes:

1. **Inline `FRichCurve` keys** on `UNiagaraDataInterfaceCurve` /
   `UNiagaraDataInterfaceColorCurve` / `UNiagaraDataInterfaceVectorCurve`
   /  `UNiagaraDataInterfaceVector2DCurve` /
   `UNiagaraDataInterfaceVector4Curve`. Stored as one or more
   `FRichCurve` members on the DI object.
2. **`UCurveFloat` / `UCurveLinearColor` asset binding** via a separate
   curve-asset DI variant (the "use external curve asset" path).

`niagara.set_property` can in theory write either shape — the generic
property path serializes JSON into the curve's `FRichCurve` member or
into a soft-object-pointer field — but in practice:

- No typed entry point exists. Agents must hand-roll the JSON for every
  key tuple (`Time`, `Value`, `InterpMode`, `ArriveTangent`,
  `LeaveTangent`, `TangentWeightMode`, `ArriveTangentWeight`,
  `LeaveTangentWeight`) and the exact property path inside the DI
  (`Curve.Keys` vs `CurveRed.Keys` / `CurveGreen.Keys` / `CurveBlue.Keys`
  / `CurveAlpha.Keys`, etc.).
- The asset-binding variant (DI flavor that points at a `UCurveFloat`
  or `UCurveLinearColor` asset) is undocumented — agents don't know
  which property name to assign and whether the DI auto-detects the
  bound asset's channel count.
- Each curve type has a different shape (1 float curve, 3 float curves
  for vector, 4 for color, etc.), so even a working JSON-blob recipe
  doesn't carry across DI flavors.

Confirmed by:

- `call("niagara.set_curve_keys")` → NOT FOUND
- `call("niagara.bind_curve_asset")` → NOT FOUND
- grep across `Handlers/Niagara/` finds zero `Curve` / `FRichCurve` /
  `UNiagaraDataInterfaceCurve` references.

## Why it matters

Curve authoring is the backbone of polished VFX. Without typed helpers,
trail-fade, color-over-life, and force-curve effects can't be authored
through the MCP at all without the agent reverse-engineering the DI's
property layout per flavor, then emitting raw `set_property` JSON. That
defeats the purpose of the typed niagara surface.

## Proposal

Two RPCs:

```
niagara.set_curve_keys(
    assetPath, scope, parameterName,
    channel?: "x"|"y"|"z"|"w"|"r"|"g"|"b"|"a",   // omit for scalar DI
    keys: [
        {
            time: number,
            value: number,
            interp?: "constant"|"linear"|"cubic",
            arriveTangent?: number,
            leaveTangent?: number,
            arriveTangentWeight?: number,
            leaveTangentWeight?: number,
            tangentWeightMode?: "none"|"arrive"|"leave"|"both"
        }
    ],
    compile?: boolean,
    save?: boolean
) -> { written: number, channel: string }
```

Writes the supplied keys onto the matching `FRichCurve` member of the
parameter's bound `UNiagaraDataInterfaceCurve` family. Dispatches on the
DI's runtime class to pick the right member (`Curve` for scalar,
`CurveRed/Green/Blue/Alpha` for color, `CurveX/Y/Z[/W]` for vector
variants). `channel` is required for non-scalar variants; omitted means
"scalar DI".

```
niagara.bind_curve_asset(
    assetPath, scope, parameterName,
    curveAssetPath: string,    // path to UCurveFloat or UCurveLinearColor asset
    save?: boolean
) -> { bound: true, curveAsset: string, dataInterfaceClass: string }
```

Switches the parameter's DI to the asset-binding flavor (or assigns the
asset path if the DI already supports both modes) and points it at the
given curve asset. Rejects with `INCOMPATIBLE_CURVE_ASSET` if the asset
shape doesn't match the DI's channel count (e.g. `UCurveFloat` into a
color-curve DI).

Both reuse the established `FNiagaraSystemViewModel` cache so any open
editor refreshes, and reuse the `scope` resolver (`system` /
`emitter:<name>` / `module:<entryId>`) already used by
`niagara.set_module_input`.

## Cross-ref

- `niagara.set_property` — generic property path; this ticket adds the
  typed layer above it.
- `niagara.set_module_input` — peer typed authoring RPC; the
  `scope`/`target` resolver pattern is shared.
- `B-asset-dump-rawcurvedata-text-blob` — adjacent area (dump-side
  curve readability); not blocking.

## History
- `#1-no-curve-helpers` `OPEN` reporter — Confirmed `niagara.set_curve_keys` and `niagara.bind_curve_asset` don't exist; grep finds zero curve/DI-curve references under `Handlers/Niagara/`. Agents must hand-roll `FRichCurve` JSON per DI flavor (scalar / color / vector / vector2D / vector4) and figure out the right property name (`Curve.Keys` vs `CurveRed.Keys` etc.) without docs. Asset-binding DI variant is undocumented. Proposes typed `set_curve_keys` with optional `channel` for multi-curve DIs and `bind_curve_asset` for the `UCurveFloat`/`UCurveLinearColor` reference case, both reusing the existing `scope` resolver and view-model cache.
- `#2-curve-keys-and-asset-binding-rpcs` `IN-REVIEW` developer — Added NiagaraCurveHandler.cpp with niagara.set_curve_keys (dispatches on DI class to pick the right FRichCurve member; uses engine suffix-form names XCurve/RedCurve etc., not the ticket's prefix-form) and niagara.bind_curve_asset (assigns UNiagaraDataInterfaceCurveBase::CurveAsset + SyncCurvesToAsset + UpdateLUT; no class-swap — the ticket's "separate asset-binding DI flavor" model was inaccurate, the base class already carries CurveAsset). Reuses ParseDataInterfacePayload + ResolveTarget + EndEmitterMutationScope flow from add_data_interface. Regression test asserts handler registration.
- `#3-verify-rpcs-registered` `DONE` tester — Verified: `niagara.set_curve_keys?` and `niagara.bind_curve_asset?` discovery calls both return full schemas with expected params (assetPath/scope/parameterName/keys[+channel] and assetPath/scope/parameterName/curveAssetPath respectively, both with optional emitter/compile/save). Methods registered and dispatchable.
