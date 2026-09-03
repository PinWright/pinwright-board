---
id: B-pwmodel-boolean-output-takes-slot-zero
title: "Every triangle a .pwmodel boolean produces lands on material ID 0 - a `union` block's own `material=` is silently discarded and a `subtract`'s cut walls take another part's material, on a clean compile with no diagnostic"
status: OPEN
severity: High
category: bug
tags: [pwmodel, materials, slot, material-id, boolean, union, subtract, bevel, silent-wrong, no-diagnostic]
encounters: 4
lastSeen: 2026-09-03T04:35:00Z
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

- `#2-blast-radius-across-one-project-46-documents` `OPEN` reporter - Scanned every `.pwmodel` in the host project (46 documents across three streams) to size the defect beyond the two weapons in `#1`. **The `bevel` route alone silently damages 15 further models authored by a different team on the same day, none of whom knew.** All 34 environment documents carry a `materials { }` block, all 38 of their `bevel` ops omit `infer_material_id`, and 15 are multi-slot so the bleed is visible geometry rather than a no-op:
```
SM_ENV_Truck          6 slots  5 bevels  2 subtracts   first-use slot 0 = Rubber
SM_ENV_RoofPlant      4 slots  2 bevels
SM_ENV_DoorPersonnel  3 slots  3 bevels  1 subtract
SM_ENV_RoofHatch      3 slots  2 bevels  1 subtract
SM_ENV_RoofLight      3 slots  2 bevels  2 subtracts
+ 10 two-slot models: Barrier_Jersey, Cabinet, Chair, Container20, Desk,
  LightFixture, PipeSupport, RollerDoor, RubblePile, SignPanel
```
The truck is the sharpest case and the best argument for flipping the default: its first-use slot is `Rubber`, so every bevelled edge on its paint, chassis, grille and glass renders as tyre rubber. **Nobody filed a bug about any of these**, because the failure mode is "the material looks slightly wrong along the edges" rather than an obvious fault, and every diagnostic an author can reach - `diagnosticSummary`, `materialSlotList`, `unboundSlots`, `static_mesh.describe` - reports the model as correct. That is the whole shape of this defect: it is not that authors get it wrong, it is that the correct-looking authoring produces wrong triangles and nothing in the toolchain can say so.
This strengthens the `bevel` recommendation in the body from a nicety to the highest-value single change: **default `infer_material_id` to true**. 38 of 38 bevels written across three independent streams omitted it, which is the definition of a default set the wrong way round - not one author in the project chose the current behaviour. `material_id=0` remains available for the case that genuinely wants a fixed slot.

- `#3-self_union-is-a-working-in-format-remedy-for-route-1-measured-on-two-shipped-documents` `OPEN` reporter — Route (1) has an in-format fix and it is `self_union`. `self_union` is dispatched at `PwModelCompiler.cpp:2643` as a MODIFIER, not through `RunBoolean`, so it never sets `bNested` and never touches material IDs: siblings written at part level reach `RunGenerator` with `bNested=false`, take their `material=` through `ResolveSlot`, and the resolve then merges the crossing shells while every triangle keeps the slot it arrived on. The pattern that replaces `union { A ; union { B } }` is therefore **plain siblings, each carrying its own `material=`, then ONE `self_union` before the next `subtract`** — the resolve has to precede the cut, because a boolean against a mesh whose shells still cross uses a surface that bounds nothing.

  Measured end to end on `Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel`, which had eleven `union` blocks and eleven `subtract` blocks and **not one `material=` tag on any of them**. Per-triangle read-back off the baked asset with `GeometryScript_Materials.get_triangle_material_id`, before:
```
id 0 Slide   3775 tris  x[ -2.37.. 20.40]  z[-11.09..  3.95]
id 1 Frame    966 tris  x[ -2.11..  8.54]  z[-10.57..  0.41]
id 2 Sights    73 tris  x[  0.80.. 19.90]  z[  2.80..  3.95]
```
  `Slide` (`MI_WPN_MetalPhosphate`) reaching z -11.09 is the grip's butt: the polymer frame, all 110 checkered grip studs, the magazine floorplate, the rail tang and the trigger's safety tab were rendering as phosphated steel. After hoisting every keep-geometry block to siblings + `self_union`, adding `infer_material_id=true` to all ten bevels, and splitting the slide into its own document so slot 0 becomes `Frame`:
```
SM_WPN_Pistol        id 0 Frame     1928  x[-2.37..14.00]  z[-11.09.. 1.40]
                     id 1 Controls   152  x[ 5.70..11.40]  z[ -3.30.. 1.95]
SM_WPN_Pistol_Slide  id 0 Slide     2066  x[-0.60..20.40]  z[  0.55.. 3.94]
                     id 1 Sights     120  x[ 0.80..19.90]  z[  2.80.. 3.95]
```
  `health` stayed `isClosed true / boundaryEdges 0 / degenerateTriangles 0 / selfIntersections 0` on both documents through the whole change, which is the point worth carrying: it stayed clean **before** the fix too. Nothing in `health`, `materialSlotList`, `unboundSlots`, `diagnosticSummary` or `static_mesh.describe` moved when 2,800 triangles changed material. The per-triangle read-back is still the only instrument that sees this.

  **Route (2) really is unfixable and now has a number.** One `subtract` survives in the split document — the rear sight's notch — and its opened walls measure exactly **12 triangles** on slot 0 (`x 0.83..1.96, z 3.50..3.94`), phosphate inside an anodised block. The only lever left is document-wide: slot 0 is whichever part tags first, so **part order is the material of every cut wall in the document**. That is a real authoring rule nothing states — `SM_WPN_Pistol` had to keep `slide` first while the slide's eight cuts lived there, and had to have `frame` first the moment they left, or the entire inside of the trigger guard turned phosphate. An author who reorders two parts for readability silently repaints every cut face in the file.

  Suggestion, on top of `#2`'s `infer_material_id` default flip: let `subtract` take `material=` and apply it to the opened walls only. The tool is discarded, but the walls are *output*, and they are the one thing the author can name. Failing that, `PWMODEL_MATERIAL_ON_BOOLEAN` should name the part-order consequence instead of only refusing the tag.

- `#4-the-second-shipped-document-and-an-in-format-remedy-for-route-2-through-bores` `OPEN` reporter — Applied `#3`'s sibling + `self_union` pattern to the other weapon, `Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel`, which is the 21,420-triangle model this ticket's body measures. Per-triangle read-back off the baked asset, before and after (the magazine's 314 triangles left for `SM_WPN_AR_Magazine` in the same change, so the after column is the rifle alone):

```
                 BEFORE                                  AFTER
id 0 Receiver    18559  x[-26.95.. 55.00]                17059  x[-26.20.. 54.20]
id 1 Polymer       460  x[-25.30.. 13.98]                  428  x[-26.95..  1.99]
id 2 Barrel       2017  x[-23.70.. 15.50]                 4751  x[-23.70.. 55.00]
id 3 Optic         384                                     384
                                          SM_WPN_AR_Magazine  id 0 Polymer 314
```

  `Barrel` more than doubles and its x range finally reaches the flash hider at 55.00 instead of stopping at the chamber at 15.50. **The residue is exactly accountable**, which is worth recording because it makes route (2)'s cost a number rather than an impression. Summing the compile response's per-part triangle counts by the part's declared slot and differencing against the read-back: Receiver +285, Polymer -128, Barrel -157, Optic 0. The 128 are `part stock`'s bore walls and the 157 are the flash hider's six port walls — every one of them a `subtract` opening, and nothing else in a 19-part document is misplaced by a single triangle.

  **ROUTE (2) HAS AN IN-FORMAT REMEDY WHERE THE CUT IS A THROUGH-BORE, and `#3`'s "really is unfixable" needs that qualification.** A bore subtracted with `subtract { cylinder }` has untaggable walls; the SAME bore generated with `pipe outer_radius=... inner_radius=...` is ordinary generated geometry that takes the generator's `material=` like any other face. The barrel's forward three steps were rewritten that way — `cylinder` → `pipe` at 0.78/0.42, 0.92/0.68 and 1.15/0.68 — and the two `subtract` bore cylinders deleted outright. Identical solid, identical bore diameters and step station, and the muzzle crown (the one surface a player looks straight down the axis of) is now phosphate instead of anodised. Successive pipes of different inner radius union into a stepped bore correctly: the smaller inner radius wins in the overlap, so the 0.42 barrel bore steps up to the 0.68 device bore exactly where the two pipes stop overlapping.

  The remedy does NOT extend to a slot or a window — there is no generator whose output is a tube with six ports in it — so the flash hider's 157 triangles stay. **The rule worth publishing is: if the cut goes all the way through and is a surface of revolution, generate it; otherwise part order is still the only lever.**

  **THE WORKAROUND'S FILTER-BOX COST IS LARGER THAN THE BODY SAYS, AND IT IS NOT ARITHMETIC — IT BREAKS THE MESH.** The body notes that hoisting a `union { gen ; bevel }` block changes the bevel's scope and that `filter_box_min`/`filter_box_max` restores it "computed by hand per site". At 3 of 14 sites on this model no correct box exists at all, and taking the obvious one produced an OPEN, mis-wound, self-intersecting part on a compile that still reported `success: true`:

```
part bolt_catch  tab + paddle bevels, boxes at each solid's extent +0.12
                 -> 86 self-intersections, 3 boundary edges, 3 shells, orientationConsistent false
part magazine    floorplate bevel, box at its extent +0.12
                 -> 53 self-intersections, 34 boundary edges, 10 non-manifold vertices
part trigger     no separating box exists in either ordering; bevelling after the resolve instead
                 -> 82 self-intersections at the blade-to-curl junction
```

  The mechanism is the same one in all three and it is specific to this workaround: the FIRST solid has already been bevelled, so it carries a **corner patch** — the little facet where three chamfer strips meet — roughly `distance` across at each of its box corners. Any filter box drawn around a SMALL second solid contains one of those patches, and a 0.12 bevel on a 0.12 facet inverts it. Two of the three were recoverable and the third was not:

  - **Generate the larger solid first.** Whichever solid comes second is the one that needs the box, and a box around the smaller contains the larger's arrises while a box around the larger does not contain the smaller's. The magazine went from 53 crossings to 0 purely by emitting its 3.8 x 5.0 floorplate before its 3.2 x 4.6 body.
  - **Move the solid to its own `part`.** A new part starts with an empty op list, so its bevel is scoped by construction. Used for the rifle's buttpad, where every axis-aligned box holding the pad's twelve arrises also held the stock bore's rear mouth rim.
  - **Give up the edge break.** `bolt_catch`'s tab and paddle and `trigger`'s finger curl now have none, because no box and no ordering separates them. That is a visible authoring regression caused entirely by working around this ticket.

  Two smaller things this change surfaced. **`PWMODEL_UNUNIONED_OVERLAP` fires on the recommended pattern**: the warning is raised when the sibling is appended and cannot see that a `self_union` follows two lines later, so adopting the workaround took this document from 2 warnings to 10, all of them the fix being mistaken for the bug. If the fix lands as authoring guidance rather than a compiler change, that warning needs to look ahead for a `self_union` in the same op list. And **there is no `set_material_id` op in the format** — `model.describe_ops` lists `set_vertex_color` for colour and nothing at all for material id — so a part cannot re-tag itself after a boolean, which is why every remedy here has to be structural.
