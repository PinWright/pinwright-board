---
id: B-pwmodel-boolean-output-takes-slot-zero
title: "Every triangle a .pwmodel boolean produces lands on material ID 0 - a `union` block's own `material=` is silently discarded and a `subtract`'s cut walls take another part's material, on a clean compile with no diagnostic"
status: OPEN
severity: High
category: bug
tags: [pwmodel, materials, slot, material-id, boolean, union, subtract, bevel, silent-wrong, no-diagnostic]
encounters: 1
lastSeen: 2026-09-03T03:37:04Z
---

# A boolean's output is untagged, so it resolves to whichever slot the first part in the document opened

`RunBoolean` builds the tool mesh with `bNested=true`
(`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:2925`), and `RunGenerator`'s only
tagging site is gated on `if (!bNested)` (`:1681`). So a generator written inside a boolean block
never reaches `ResolveSlot`, its triangles keep `MaterialID == 0`, and `GeometryOps::Boolean`
(`:2975-2976`) carries them straight into the result at 0 - where 0 is not a neutral default but
**whatever the first part in the document tagged** (`:530-531`, `:561-569`).

Three separate routes onto slot 0, measured together below:

| route | what lands on slot 0 |
|---|---|
| `union { gen material="X" }` | **all** of the unioned geometry; the inner `material=` parses and is discarded |
| `subtract { tool }` | the walls the cut opens (base geometry keeps its slot) |
| `bevel` without `infer_material_id=true` | every new bevel face - the op defaults `infer_material_id=false`, `material_id=0` |

The `union` route is the sharp one: the author *wrote the tag*, on a generator, in an op the parser
accepts it on, and it silently does nothing. `PWMODEL_MATERIAL_ON_BOOLEAN` fires only for
`material=` on the **boolean op itself**, so the spelling that is actually written raises nothing.

## Measured

`X:/src/unreal/EAContentExamples58/Content/FPS/Weapons/Meshes/Test/SM_WPN_SlotProbe.pwmodel`, seven
parts, five slots, compiled to `/Game/FPS/Weapons/Meshes/Test/SM_WPN_SlotProbe` and read back per
triangle off the baked asset with `GeometryScript_Materials.get_triangle_material_id`. Slot 0 is
`Alpha`, opened by `part_a`; every other part tags only its own slot.

```
part_a  base(x0)      -> id 0 (Alpha  )  tris=12     part_a is the only legitimate user of slot 0
part_a  unioned(x20)  -> id 0 (Alpha  )  tris=12
part_b  base(x0)      -> id 1 (Beta   )  tris=12
part_b  unioned(x20)  -> id 0 (Alpha  )  tris=12  <-- union { box material="Beta" } lost its tag
part_c  side wall     -> id 2 (Gamma  )  tris=8
part_c  cap (poked)   -> id 2 (Gamma  )  tris=40      a subtract KEEPS the base geometry's slot
part_c  bore wall     -> id 0 (Alpha  )  tris=64  <-- the walls the cut opened
part_c  unioned(x20)  -> id 0 (Alpha  )  tris=12
part_d  base(x0)      -> id 3 (Delta  )  tris=28      self_union PRESERVES tags - see workaround
part_d  sibling(x6)   -> id 3 (Delta  )  tris=14
part_e  side wall     -> id 4 (Epsilon)  tris=8
part_e  cap (poked)   -> id 4 (Epsilon)  tris=40
part_e  bore wall     -> id 0 (Alpha  )  tris=64
part_f  bevel, no infer  -> id 0 (Alpha) tris=32  <-- 32 of a beveled cube's 44 triangles
part_f  bevel, no infer  -> id 5 (Zeta ) tris=12
part_g  bevel infer=true -> id 6 (Eta  ) tris=44      all 44, correct
```

`diagnosticSummary` on that compile: `errors: 0`. The only warning is `PWMODEL_FLOATING_COMPONENT`,
about the probe's deliberate layout. Nothing names the material damage.

## What it costs on a real model

`X:/src/unreal/EAContentExamples58/Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel` - 18 parts, 4
slots, every part correctly tagged in source, `static_mesh.describe` reporting the right material
per slot. Read back per triangle:

```
slot 0 Receiver (MI_WPN_MetalAnodised)   18559 tris   x[-26.95..55.00]
slot 1 Polymer  (MI_WPN_Polymer)           460 tris
slot 2 Barrel   (MI_WPN_MetalPhosphate)   2017 tris   x[-23.70..15.50]
slot 3 Optic    (MI_WPN_OpticGlass)        384 tris
```

**87% of the rifle ships on the first part's material.** The barrel runs x 11.5..55.0 and only its
first cylinder - the one written outside a `union` block - carries `Barrel`; the other four steps,
the gas block and every bevel render as anodised receiver aluminium. The companion pistol has the
same shape with the polarity reversed: slot 0 is `Slide` (phosphated steel), so its polymer frame
and all 110 checkered grip studs, every one of them union output, render as steel.

Both were reported by a reviewer as "the compiled slot assignment is scrambled". It is not
scrambled - the slot **table** is right, `materialSlotList` is right, and the source tags are right.
Only the triangles are wrong, and no field in the compile response shows triangles per slot.

## Workaround, and why it is not a fix

`self_union` (`part_d` above) is **not** routed through `RunBoolean` and **preserves** material IDs.
So `union { gen material="X" }` can be rewritten as a plain sibling `gen material="X"` followed by
one `self_union` at the end of the run, and the tag survives. That covers the `union` route. It does
not cover a `subtract`'s cut walls, which have no op of their own to tag and stay at 0 - leaving
"order the document so slot 0 is the material most cut interiors should show", the same
undiscoverable part-ordering workaround `B-pwmodel-modifier-output-takes-slot-zero` records for
`spiral_stair`.

Rewriting `union` as sibling + `self_union` also silently changes the scope of any modifier that was
inside the block: a `bevel` written after the generator in a `union { }` bevels only the tool, while
the same `bevel` hoisted to part level bevels the whole accumulated part. `filter_box_min` /
`filter_box_max` is the only way to restore the scope, and it has to be computed by hand per site.

## Fix

Tag boolean output the way `#4` on `B-pwmodel-modifier-output-takes-slot-zero` tags additive
modifier output: snapshot the triangle set before the op and assign a slot only to triangles the op
appended, so nothing already tagged is recoloured.

- **`union` / `intersection` / `trim`:** the tool's own ops know their slots. Resolve them at part
  level (drop the `!bNested` gate for a *tool* whose result is kept, or resolve the tool's slot
  names into the model-wide table without allocating, then remap after the boolean). An untagged
  tool inherits the slot of the geometry it was unioned onto, as the modifier fix does.
- **`subtract`:** the new walls should inherit the slot of the geometry they were cut into - that is
  unambiguous where the cut geometry is single-slot, and `PWMODEL_MODIFIER_MATERIAL_AMBIGUOUS`
  already exists for the mixed case.
- **`bevel`:** flip `infer_material_id` to default **true**. New faces taking the two faces either
  side of the edge is right in every case an author writes a bevel for; `material_id=0` as the
  silent default is right in none. This is a defaults change, not a capability gap - the parameter
  works today (`part_g`).
- Whatever lands, a warning when a boolean emits triangles into a slot the enclosing part never
  bound is the part that makes it visible. The compile response should also carry triangles per
  slot; `materialSlotList` cannot show this class of fault, and `static_mesh.describe` cannot
  either.

## Related

- `B-pwmodel-modifier-output-takes-slot-zero` (IN-REVIEW) - the same failure for `sweep` /
  `extrude_along_spline`. Its `#4` fix explicitly stays on the `!bNested` modifier path and cites
  `D-03` as the boolean branch, so it does not cover this. The `SnapshotModifierMaterial` /
  `ApplyModifierMaterialTag` pair it adds is the mechanism this needs.
- `Docs/plans/defect-backlog.md` `D-03` (REFINED) - slot allocation skipped on the `!bNested`
  boolean/hull branch. That is this branch, described as a slot-*allocation* question; this ticket
  is the triangle-*tagging* consequence and the measured cost.
- `B-pwmodel-untagged-generator-inserts-default-slot` - the adjacent case where an untagged
  generator opens `Default` rather than silently taking slot 0.

## History
- `#1-boolean-output-and-bevel-faces-land-on-slot-zero` `OPEN` reporter - `RunBoolean` builds its tool with `bNested=true` (`PwModelCompiler.cpp:2925`) and `RunGenerator`'s only tagging site is gated `if (!bNested)` (`:1681`), so a generator inside a boolean block never reaches `ResolveSlot` and its triangles keep `MaterialID == 0` - which is the first part's slot (`:530-531`, `:561-569`), not a neutral default. Measured on a seven-part five-slot probe compiled to an asset and read back per triangle: `union { box material="Beta" }` lands entirely on `Alpha`; a `subtract`'s bore walls (64 tris) land on `Alpha` while the base box keeps its own slot; `bevel` without `infer_material_id=true` puts 32 of a beveled cube's 44 triangles on `Alpha`. Clean compile, `errors: 0`, no diagnostic. Cost on a real 18-part rifle: 18559 of 21420 triangles on the first part's material - the barrel, gas block and every bevel render as receiver aluminium, and the companion pistol's polymer frame and 110 grip studs render as phosphated steel, because that document's slot 0 is the slide. `materialSlotList`, `static_mesh.describe` and the source tags are all correct; nothing in any response reports triangles per slot, so the fault is invisible until the asset is rendered. `self_union` is not routed through `RunBoolean` and preserves IDs, which makes sibling + `self_union` a workaround for `union` only - and it changes the scope of any modifier that was inside the block, recoverable only by hand-computed `filter_box_min`/`filter_box_max`. Fix: tag boolean output using the snapshot-and-retag mechanism `#4` on `B-pwmodel-modifier-output-takes-slot-zero` adds for modifiers; have `subtract` walls inherit the cut geometry's slot; default `bevel infer_material_id` to true; warn when a boolean emits into a slot the enclosing part never bound; and report triangles per slot on `model.compile`.
