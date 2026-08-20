---
id: B-pwmodel-modifier-output-takes-slot-zero
title: "Geometry produced by a .pwmodel modifier keeps material ID 0, so it lands on whichever slot the FIRST part in the document tagged — one part ships in two materials, with no way to tag it and no diagnostic"
status: OPEN
severity: High
category: bug
tags: [pwmodel, materials, slot, material-id, modifier, sweep, extrude_along_spline, cross-part, silent-wrong, no-diagnostic]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# Slot 0 is another part's material, and modifier output always takes it

`.pwmodel` material slots are a **model-wide** table allocated in first-use order, and the table
index *is* the material ID written onto triangles —
`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:530-531`:

```cpp
// Model-wide, first-use order; the index IS the material ID written onto triangles.
TArray<FString> SlotNames;
```

`ResolveSlot` (`:561-569`) appends on first use, so **slot 0 is whatever the first part in the
document tagged** — not a neutral default.

Tagging happens in exactly one place: `RunGenerator`, `PwModelCompiler.cpp:1014-1036`, which calls
`ClearMaterialIDs(Scratch, ResolveSlot(SlotName), nullptr)` at `:1034-1035` behind `if (!bNested)`.
Modifiers take the other branch — `RunOp` dispatches to `DispatchModifier` at `:2066` before ever
reaching `RunGenerator` at `:2076` — and `DispatchModifier` contains no `ResolveSlot` call anywhere.
`sweep` (`:1792-1824`) and `extrude_along_spline` (`:1825-1849`) build their params and call straight
through with no material handling at all, so their triangles keep
`FGeometryScriptPrimitiveOptions::MaterialID == 0` and resolve to the first document-wide slot.

## The author cannot correct it

`MakeModifier` (`Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:154-157`) does **not** call
`AcceptMaterialSlot`, while `MakeGenerator` (`:143-152`) does. `sweep` (`PwModelParser.cpp:996`) and
`extrude_along_spline` (`:1010`) are registered through `MakeModifier`, so writing `material=` on
them is rejected as an unknown parameter. The parser's own comment at `PwModelParser.cpp:864-870`
records this identical hole already found once for `append_triangle` and fixed there by adding
`AcceptMaterialSlot` (`:1019`) — the same fix was never applied to the modifiers.

## Nothing reports it

`PwModelDiagnostic.h` carries `PWMODEL_MATERIAL_ON_BOOLEAN`, `PWMODEL_MATERIAL_ID_CONFLICT`,
`PWMODEL_UNUSED_MATERIAL` and `PWMODEL_MATERIAL_ID_OUT_OF_RANGE` — nothing for untagged modifier
output. And because `ResolveSlot` is never called, the slot count is unchanged, so the padding path
at `PwModelCompiler.cpp:2329-2342` cannot raise `PWMODEL_MATERIAL_ID_OUT_OF_RANGE` either. Both
guards miss for the same reason.

## Measured

`Docs/wiki-src/model.vertex-color.md:72` records the probe: part `a` tags `Alpha`; part `b` tags only
`Beta` and sweeps; the swept rod comes back on **`Alpha`**. One part rendering in two materials, on a
clean compile, with an unchanged slot count. `spiral_stair` orders `part handrail` first purely to
work around this — the workaround is "reorder parts so the slot you want lands at index 0", which is
undiscoverable and breaks whenever a part is added above it.

## Not D-07

`Docs/plans/defect-backlog.md:172-188` (`D-07`, Status CONFIRMED) covers the `geometry.*` RPC verbs
lacking a `materialId` parameter — including `sweep` and `extrude_along_spline` — and its symptom is
"cannot take a different material without a separate selection pass": no *choice*. This is a
different layer and a different symptom.

| | D-07 | this |
|---|---|---|
| Layer | `geometry.*` RPC verbs, `GeometryOps_Advanced.cpp` | `.pwmodel` front end: parser op table + `RunGenerator`/`DispatchModifier` split |
| Missing thing | a raw `materialId` **integer** on the RPC | a `material=` **slot name** on the op (`AcceptMaterialSlot`, `PwModelParser.cpp:113-119`) |
| Symptom | no way to choose a material | geometry silently takes **another part's** material via first-use table ordering |
| Fix | set `PrimOptions.MaterialID` from a new param | route modifier output through `ResolveSlot`, or inherit the enclosing part's slot; add a diagnostic |

D-07's fix is necessary but **not sufficient**: adding a raw `materialId` to `GeometryOps::Sweep`
still leaves the `.pwmodel` op with no `material=` parameter, no way to resolve a slot *name*, and no
diagnostic when one part's modifier output contaminates another part's slot. Closest in shape is
`D-03` (slot allocation skipped on a compiler branch), but that is the `!bNested` boolean/hull path —
a different branch.

**Fix:** have modifier output inherit the enclosing part's resolved slot rather than falling to ID 0,
and add `AcceptMaterialSlot` to `MakeModifier` so `material=` is authorable — the `append_triangle`
precedent at `PwModelParser.cpp:1019` is the pattern. Whichever lands, a `PWMODEL_*` warning for
geometry emitted into a slot the enclosing part never bound is the part that makes it visible.

## Related

- `Docs/plans/defect-backlog.md` `D-07` (CONFIRMED) — the RPC-layer half; necessary, not sufficient.
- `Docs/plans/defect-backlog.md` `D-03` (REFINED) — slot allocation skipped on the `!bNested` branch.
- `B-revolve-polygon-drops-material-id` (OPEN) — `append_revolve_polygon` ignoring
  `PrimitiveOptions.MaterialID`; an upstream-engine bug on a different verb.

## History
- `#1-modifier-output-lands-on-slot-zero` `OPEN` reporter — `.pwmodel` material slots are a model-wide table allocated in first-use order whose index IS the material ID written onto triangles (`PwModelCompiler.cpp:530-531`, `:561-569`), so slot 0 is whatever the first part in the document tagged. Only `RunGenerator` tags (`PwModelCompiler.cpp:1014-1036`, `ClearMaterialIDs(Scratch, ResolveSlot(SlotName), nullptr)` at `:1034-1035`); `RunOp` routes modifiers to `DispatchModifier` first (`:2066`), and `sweep` (`:1792-1824`) and `extrude_along_spline` (`:1825-1849`) contain no `ResolveSlot` call, so their triangles keep `FGeometryScriptPrimitiveOptions::MaterialID == 0` and resolve to another part's slot. The author cannot correct it: `MakeModifier` (`PwModelParser.cpp:154-157`) omits the `AcceptMaterialSlot` call `MakeGenerator` makes (`:143-152`), so `material=` on those ops is an unknown-parameter error — the same hole already found and fixed for `append_triangle` (`PwModelParser.cpp:864-870`, `:1019`). Nothing reports it: no `PWMODEL_*` code covers untagged modifier output, and since `ResolveSlot` is never called the slot count is unchanged, so `PWMODEL_MATERIAL_ID_OUT_OF_RANGE` (padding path `:2329-2342`) cannot fire either. Measured (`Docs/wiki-src/model.vertex-color.md:72`): part `a` tags `Alpha`, part `b` tags only `Beta` and sweeps, and the swept rod lands on `Alpha` — one part in two materials on a clean compile; `spiral_stair` orders `part handrail` first solely to work around it. Distinct from `D-07`, which asks for a raw `materialId` integer on the `geometry.*` verbs: that fix leaves the `.pwmodel` slot-*name* path and the missing diagnostic untouched, and D-03's slot-skip is the `!bNested` boolean/hull branch, not this one. Fix: inherit the enclosing part's resolved slot for modifier output, add `AcceptMaterialSlot` to `MakeModifier`, and warn when geometry is emitted into a slot the enclosing part never bound.
