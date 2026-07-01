---
id: B-decompile-struct-subfield-dropped
title: "IR decompile drops a non-default struct sub-field (pcg.decompile omits bRandomizedPruning=false): nested struct is ExportText'd with a nullptr default, so sub-fields are compared against the zero-value not the CDO default — a non-default value equal to the zero-value is silently dropped while a default value is spuriously emitted"
status: OPEN
severity: High
category: bug
tags: [pcg, decompile, ircore, reflected-emit, struct, exporttext, null-default, lossy, self-pruning]
encounters: 1
lastSeen: 2026-07-01T09:47:48.1344041+03:00
---

# IR decompile exports nested structs against the *zero-value*, not the CDO default — non-default sub-fields equal to zero are dropped, default sub-fields are spuriously emitted

`pcg.decompile` (and, via the same shared IrCore helper, every reflected-field
IR decompiler) emits a struct-valued reflected property by exporting the whole
struct through `FProperty::ExportTextItem_InContainer(..., DefaultValue = nullptr, ...)`.
With a `nullptr` default, each **sub-field**'s default-suppression is computed
against the property type's **zero value**, not against the struct's real CDO
default. Consequences, both wrong:

1. A sub-field set to a **non-default** value that happens to equal its type's
   **zero value** is **silently dropped** from the export. Concretely, on a
   `UPCGSelfPruningSettings` node, `bRandomizedPruning` whose real default is
   `true` (`PCGSelfPruning.h:48`), when set to `false`, is omitted entirely —
   the decompiled text represents the node as if the knob were still at its
   default `true`.
2. A sub-field left at a **non-zero default** is **spuriously emitted** as if it
   were overridden (e.g. the default `ComparisonSource=$Extents` and
   `CollisionAttribute=@Last` selectors appear in the export even though they are
   the CDO defaults; `RadiusSimilarityFactor=0.250000` would likewise appear if
   left at its default).

The net effect is the exact **inversion** of correct decompile behaviour for
`bRandomizedPruning`: setting it to its **default** (`true`) makes it **appear**,
and setting it to a **non-default** (`false`) makes it **disappear**. Since
`pcg.decompile` is sold as a faithful, diffable text export (paste into a design
doc, diff against future revisions — the exact task here), a reviewer diffing the
export would see **no change** when the author toggled the knob off, and would
see a spurious `RadiusSimilarityFactor=0.25` "change" that never happened. The
caller trusts a lie.

This is **not PCG-specific**: the offending export lives in the shared IrCore
helper `FIrTextUtils::FormatReflectedPropertyValue`, which MGIR
(`material.decompile_mgir`) and AGIR (`anim.decompile_agir`) also route their
struct-valued property emit through (per `F-ircore-reflected-property-emit`), so
any IR decompiler emitting a struct property whose sub-field's non-default value
coincides with the zero value drops that sub-field the same way.

## Root cause (guilty source lines)

`Plugins/PinWright/Source/PinWright/Private/IrCore/IrTextUtils.cpp:673-675`
— the struct fallback in `FormatReflectedPropertyValue` passes `nullptr` as the
`DefaultValue`:

```cpp
    FString Exported;
    Property->ExportTextItem_InContainer(Exported, Container, nullptr, OwnerForExportText, PPF_None);
    return Quote(Exported);
```

For an `FStructProperty`, `UScriptStruct::ExportText` with a null `Defaults`
compares each member against a null default, which the per-property `Identical`
implementations treat as the **zero value** (e.g. `FBoolProperty::Identical(A, nullptr)`
compares `A` against `false`). So `bRandomizedPruning=false` == zero-value → the
member is suppressed; `bRandomizedPruning=true` != zero-value → the member is
emitted.

This disagrees with the walker's own **top-level** suppression, which correctly
uses the real CDO default —
`IrTextUtils.cpp:711-719` in `AppendReflectedFields`:

```cpp
    for (FProperty* Property : Properties)
    {
        if (Default && Property->Identical(
                Property->ContainerPtrToValuePtr<void>(Instance),
                Property->ContainerPtrToValuePtr<void>(Default),
                PPF_None))
        {
            continue;
        }
```

So the whole `Parameters` struct is correctly *included* (it differs from the
CDO overall), but its *inner* sub-fields are then filtered against the zero value
instead of the CDO's per-sub-field defaults — dropping the very sub-field
(`bRandomizedPruning`) whose change caused the struct to be included in the first
place. `PCGIRDecompiler.cpp` (`AppendSettingsProperties`, lines 71-101) passes
the CDO (`SettingsClass->GetDefaultObject()`) as `Default` for the top-level
check, but that default never reaches the struct's inner `ExportTextItem` call.

## What it should do

When exporting a struct-valued reflected property, pass the **matching sub-object
of the default/CDO** (`Property->ContainerPtrToValuePtr<void>(Default)`) as the
`DefaultValue` to `ExportTextItem_InContainer` so `UScriptStruct::ExportText`
suppresses sub-fields against their real archetype defaults — not the zero value.
Then `bRandomizedPruning=false` (non-default) is emitted, `bRandomizedPruning=true`
(default) is suppressed, and the default `ComparisonSource`/`CollisionAttribute`
selectors stop appearing spuriously. `FormatReflectedPropertyValue` currently has
no access to the `Default` container — plumb it through from `AppendReflectedFields`
(which already holds it) into the struct-export path.

## Verbatim repro (replay-confirmed at HEAD)

Graph `/Game/PCG/PCG_RockScatter` with a `UPCGSelfPruningSettings` node
`SelfPruning_0` (decompiled local id `N3`).

1. Set the knob to a **non-default** value (real default is `true`):
   `pcg.set_self_pruning_settings { graphPath:"/Game/PCG/PCG_RockScatter", nodeId:"SelfPruning_0", pruningType:"LargeToSmall", radiusSimilarityFactor:0.5, bRandomizedPruning:false }` → `{"nodeId":"SelfPruning_0"}` (write accepted).
   `pcg.decompile { assetPath:"/Game/PCG/PCG_RockScatter" }` → N3:
   ```
   Parameters = "(ComparisonSource=PCGBegin($Extents)PCGEnd,RadiusSimilarityFactor=0.500000,CollisionAttribute=PCGBegin(@Last)PCGEnd)"
   ```
   **`bRandomizedPruning` is absent** — the non-default `false` was dropped.

2. Set the same knob to its **default** value:
   `pcg.set_self_pruning_settings { ..., bRandomizedPruning:true }` → `{"nodeId":"SelfPruning_0"}`.
   `pcg.decompile { assetPath:"/Game/PCG/PCG_RockScatter" }` → N3:
   ```
   Parameters = "(ComparisonSource=PCGBegin($Extents)PCGEnd,RadiusSimilarityFactor=0.500000,bRandomizedPruning=True,CollisionAttribute=PCGBegin(@Last)PCGEnd)"
   ```
   **`bRandomizedPruning=True` now appears** — the default value was emitted.

The inversion (default shown, non-default hidden) is the smoking gun. The default
selectors `ComparisonSource=$Extents` / `CollisionAttribute=@Last` appearing in
both exports are the spurious-emit half of the same root cause.

severity rationale: impact=silent-wrong/lossy readback on a normal path (decompile hides a set field and shows an unset default one — the caller diffs the export and trusts the lie) × reach=shared IrCore struct-emit across MGIR/AGIR/PCG decompilers, triggered by any non-zero-default sub-field (bools defaulting true are common) -> High

## History
- `#1-initial-repro` `OPEN` reporter — Seed `pcg.decompile` (SEED mode); realistic rock/foliage scatter authoring task, then a paste-and-diff export. Replay-confirmed at HEAD via two `pcg.decompile` calls on `/Game/PCG/PCG_RockScatter` node `SelfPruning_0`: with `bRandomizedPruning=false` (non-default; real default `true` per `PCGSelfPruning.h:48`) the field is dropped from the exported `Parameters` struct; with `bRandomizedPruning=true` (default) the field is emitted — the exact inversion of correct decompile suppression. Root cause: `FIrTextUtils::FormatReflectedPropertyValue`'s struct fallback (`IrCore/IrTextUtils.cpp:674`) calls `ExportTextItem_InContainer(..., DefaultValue=nullptr, ...)`, so `UScriptStruct::ExportText` suppresses sub-fields against the zero-value, not the CDO default the top-level `AppendReflectedFields` check (`IrTextUtils.cpp:711-719`) uses — the two disagree, so the struct is included but its changed bool sub-field is filtered out. Shared helper → also affects MGIR/AGIR struct-property emit (`F-ircore-reflected-property-emit`). Deduped with ripgrep over OPEN+closed board: `F-pcg-decompile-ir` (DONE, the feature that shipped `pcg.decompile` — no fidelity carve-out for nested-struct defaults), `E-pcg-set-self-pruning-enum-undiscoverable` (docs/discoverability of the setter enum, not decompile), `F-ircore-reflected-property-emit` (DONE refactor that created the shared helper — didn't spec sub-field default handling), `B-agir-alphaboolblend-empty-struct-not-reimportable` (AGIR **compile**-side unquote, opposite direction), `B-instanced-struct-export-opaque` (`FInstancedStruct` type-erased JSON export, different surface), `B-properties-struct-export-text-fallback` (property.get/asset.dump JSON, different surface) — none covers the decompile emit-side zero-value-vs-CDO struct default bug. Fix: plumb the matching default sub-object into the struct-export path so sub-fields suppress against their real archetype defaults.
</content>
</invoke>
