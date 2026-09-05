---
id: B-pwmodel-boolean-output-takes-slot-zero
title: "Every triangle a .pwmodel boolean produces lands on material ID 0 - a `union` block's own `material=` is silently discarded and a `subtract`'s cut walls take another part's material, on a clean compile with no diagnostic"
status: IN-REVIEW
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

## Fix as landed

**Verified TRUE against source before changing anything**, and the mechanism is exactly as
reported: `RunBoolean` ran its block with `bNested=true`, `RunGenerator`'s only tagging site was
gated `if (!bNested)`, and `RunOp`'s modifier tagging carried the same gate - so nothing inside a
boolean block ever reached `ResolveSlot`. `FBevelParams` defaults `bInferMaterialID = false,
SetMaterialID = 0` (`GeometryOps_Modeling.h:569-570`), which is the third route. All three fixed.

**The design decision, and why this shape.** The output is NOT retagged after the operation. A
boolean renumbers triangles wholesale, so the snapshot-and-diff by triangle id that the modifier
fix uses is unsound here - which is why `#4` on `B-pwmodel-modifier-output-takes-slot-zero`
deliberately stopped at the modifier path. Instead **the operands are tagged before the
operation** and GeometryCore's own attribute transfer carries the ids across - which this
ticket's own measurements prove works, since the tool's material ID 0 is exactly what reached the
bore walls and the unioned box. Both operands index the one model-wide slot table, so there is no
second id space and nothing is remapped afterwards.

Faces the engine invents from NEITHER operand (a hole fill; `fill_holes` defaults true on all
four ops) cannot be carried in that way, so material ID 0 is **reserved across both operands**
for the duration of the call: every id is shifted +1 before the op and released after, and a
triangle still on 0 in the result is by construction created-here and takes the new-face slot.
That closes the last silent-zero residue rather than leaving one.

`cut_material=` was not needed and is retired as a reserved parameter. `material=` on the boolean
itself names the faces the operation creates - the only spelling a cut's wall ever has - and
`Docs/pwmodel-design.md` now records why the reservation was wrong: the answer is carried IN with
the operands, never read OUT of the result, so no component identity is involved.

**Semantics now stated normatively in `Docs/pwmodel-format.md`** (new section *What a boolean
does to material slots*):

1. Triangles the operation KEEPS carry the slot they arrived on. A boolean never recolours.
2. Ops inside the block resolve `material=` through the same model-wide table, so
   `union { box material="Beta" }` is on `Beta` and `subtract { cylinder material="Bore" }` puts
   `Bore` on the walls the cut opens.
3. `material=` on the boolean names the faces it CREATES; untagged, they inherit the slot of the
   geometry the boolean was applied to (dominant slot, ties to the lower index).

`bevel`: `infer_material_id` now defaults **true** at the document layer and `material_id`
defaults to the dominant slot of the geometry being bevelled - never a fixed 0. `bevel` also
gained `material=`. `GeometryOps::FBevelParams` is left pinned to the engine's defaults on
purpose: a parity test holds it against `FGeometryScriptMeshBevelOptions` and `geometry.bevel`
publishes it, so the divergence lives in the pwmodel dispatch where it belongs.

`color=` on a boolean stays an error (`PWMODEL_MATERIAL_ON_BOOLEAN`, message rewritten). The
asymmetry is deliberate: a boolean creates faces but no *vertices*, so a scalar colour could only
overwrite one an operand's generator already wrote.

**Two new diagnostics**, both raised by the compiler because nothing downstream can see either:
`PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS` (multi-slot target, untagged op, fallback used - or an empty
target with nothing to inherit) and `PWMODEL_BOOLEAN_MATERIAL_UNUSED` (a `material=` that opened
a slot no triangle of the result carries - the realistic source being a tool solid the operation
discards entirely, usually one that misses the target while its siblings in the block hit it, so
the op itself reports nothing wrong).

**Collision is excluded.** A `collision { hull { ... } }` body runs through the PART vocabulary,
so a boolean inside one reaches the same code. It is told apart by arriving with `bNested` set
and no enclosing boolean block, and keeps the old behaviour exactly - no slot allocated, no id
touched, no diagnostic raised.

### Files changed

- `Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp` - `CountMaterialTriangles` /
  `DominantMaterialSlot` / `ReserveMaterialIDZero` / `ReleaseMaterialIDZero` helpers;
  `FCompiler::FBooleanBlockMaterial` state + `TagBooleanBlockGeometry`; `RunBoolean` head and
  tail rewritten; `RunGenerator` and `RunOp` tagging scopes; `bevel` dispatch defaults;
  `FCompiler::DescribeSlot` extracted from a lambda.
- `Source/PinWrightGeometry/Private/Model/PwModelDiagnostic.h` - two new codes,
  `PWMODEL_MATERIAL_ON_BOOLEAN` re-documented.
- `Source/PinWrightGeometry/Private/Model/PwModelParser.h` - `FPwModelOpSpec::bSelfTagsMaterial`.
- `Source/PinWrightGeometry/Private/Model/PwModelParser.cpp` - booleans and `bevel` accept
  `material=`; a boolean rejects only `color=`; `ValidateMaterialSlots` recurses into
  `Op.Children` for tags; bevel parameter docs.
- `Source/PinWrightGeometry/Private/Tests/Model/TestPwModelBooleanMaterial.cpp` - **new**, six
  tests (union block keeps its slot; cut walls take the op's named slot; bevel faces join the
  surface they chamfer; the language surface; ambiguity reported; a tag reaching no face
  reported).
- `Source/PinWrightGeometry/Private/Tests/Model/TestPwModelParser.cpp`,
  `TestPwModelWidenedOpVocabulary.cpp`, `TestPwModelModelingOpVocabulary.cpp`,
  `TestPwModelCompiler.cpp` - updated for the language change. Two of these were premises the
  fix inverts rather than mechanical edits: `MaterialInsideBooleanBlockDoesNotBindSlot`
  asserted that a tag inside a block is NOT a reference (renamed to
  `...BlockBindsItsSlot`, now asserting both directions), and the cross-slot
  `PWMODEL_UNUNIONED_OVERLAP` case pinned the message text "use union only when both solids
  use the same material slot" - advice that was true only because a block dropped the tool's
  slot. That runtime message is rewritten too: union is now the resolution, not the hazard.
- `Docs/pwmodel-format.md`, `Docs/wiki-src/model.authoring.md`, `Docs/pwmodel-design.md`,
  `Docs/format-decisions.md`, `Docs/defect-backlog.md` (D-03).

### Reviewer verification

**NOT compiled and NOT run** - a separate compile pass follows. A reviewer should:

1. Build, then run `PinWright.Model.` and `PinWright.Geometry.Ops.` in ONE editor instance.
2. Re-compile the seven-part probe from the Measured section above and read the asset back per
   triangle with `GeometryScript_Materials.get_triangle_material_id`. Expected changes:
   `part_b unioned` on `Beta` not `Alpha`; both `bore wall` rows on their own part's slot, not
   `Alpha`; `part_f`'s 32 bevel triangles on `Zeta`, all 44 on one slot.
3. Recompile `SM_WPN_AR.pwmodel` from its **pre-workaround** revision - the sibling +
   `self_union` rewrite recorded in `#3` / `#4` should no longer be needed - and check `Barrel`
   reaches x 55.00.
4. Confirm the ordinary document stays quiet: a `subtract` into single-slot geometry must raise
   neither new code. Noise there would get the warning switched off wholesale and take the real
   case with it.

**Scope limits, deliberate.** Triangles-per-slot on the `model.compile` response is NOT
implemented: it is a response-shape change on a different surface, and the two new warnings carry
the visibility this fix needs. A slot a boolean opened that nothing carries is warned about but
NOT rolled back, so the asset still gains an empty section - rolling it back would renumber the
table, which is not worth the risk for a case that is already reported.

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
- `#5-boolean-and-bevel-output-now-tagged-at-the-operands` `IN-REVIEW` developer - Verified true against source, then fixed all three routes by tagging the OPERANDS rather than retagging the result: a boolean renumbers triangles, so the modifier fix's snapshot-and-diff is unsound here, but GeometryCore carries a surviving triangle's material id across - which this ticket's own measurements prove, since the tool's ID 0 is exactly what reached the bore walls. Ops inside a boolean block now resolve `material=` through the same model-wide slot table; `material=` is accepted on the boolean itself and names the faces the operation creates; untagged block geometry and cut walls inherit the dominant slot of the geometry the boolean was applied to. Material ID 0 is reserved across both operands for the duration of the call, so a face the engine invents from neither operand (a hole fill - `fill_holes` defaults true) is identifiable and takes the same slot instead of silently landing on the first part's material. `bevel` gained `material=` and its document-layer defaults flipped to `infer_material_id=true` with the dominant slot as the disagreement fallback; `GeometryOps::FBevelParams` stays pinned to the engine's defaults because a parity test and the `geometry.bevel` RPC both depend on them. `color=` on a boolean stays refused and the message now names `set_vertex_color`. New codes `PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS` and `PWMODEL_BOOLEAN_MATERIAL_UNUSED`. Collision hull bodies keep the old behaviour exactly. `cut_material=` is retired as a reserved parameter: it was reserved on the belief that the new faces had to be identified after the fact, and they never did. Six new tests in `TestPwModelBooleanMaterial.cpp`, three existing test files updated for the language change, language docs synced. NOT compiled and NOT run - see the reviewer verification steps above.
- `#6-trim-is-a-subtract-not-a-discard` `IN-REVIEW` developer - The post-wave suite failed one of the six new tests, `MaterialSlots.BooleanMaterialTagThatReachesNoFaceIsReported`, and the compiler was RIGHT while the fixture and three comments were wrong. The fixture used `trim material="Cap" fill_holes=false` on the belief that a trim discards its whole cutting surface, so the tag would reach nothing. `GeometryOps::Trim` maps `keep_inside` onto Subtract / Intersection and dispatches the same `ApplyMeshBoolean` the other three booleans use (`GeometryOps_Boolean.cpp:351-353`, `:407`) - so a trim's tool surface BECOMES the cut face, `Cap` landed on the flat top the trim opened, and the warning correctly stayed silent. Diagnosed from the suite log alone: the run recorded `slots [0:Shell, 1:Cap]` with a clean compile and only the two pre-existing `PWMODEL_UNBOUND_MATERIAL` warnings, which proves `RunBoolean` reached the unused-tag loop and found a result triangle carrying the slot. Fixture replaced with the case that provably produces no face - a second tool solid 500 uu from the target inside a block whose first solid does cut, so the boolean works, `PWMODEL_BOOLEAN_NO_EFFECT` stays quiet, and the dead tool's `material="Ghost"` reaches zero triangles - and the test now also asserts the slot the cut walls DO carry is not named, and that the ambiguous code does not fire alongside it. The false `trim` claim was repeated in the diagnostic's own comment (`PwModelDiagnostic.h`), the warning message text (`PwModelCompiler.cpp`), `Docs/pwmodel-format.md` (two places) and `Docs/wiki-src/model.authoring.md`; all five now name a discarded tool solid and state explicitly that trim is not one. No compiler behaviour changed.

- `#7-verified-in-fps-build` `IN-REVIEW` reviewer - **All three routes fixed. Verified per triangle off the baked asset, not off `model.compile`.** Plugin at origin/master (12 commits, 770 source files), rebuilt, wiki regenerated; UE 5.8 editor, PinWright MCP :27145.

  Call: `model.compile { filePath: "X:/src/unreal/EAContentExamples58/Content/FPS/Weapons/Meshes/Test/SM_WPN_SlotProbe.pwmodel", outputPath: "/Game/FPS/Weapons/Meshes/Test/SM_WPN_SlotProbe", save: true, diagnosticLimit: 0, diagnosticSeverity: "all", collapseDiagnostics: true }` -> `success: true`, `savedToDisk: true`, `saveState: "written"`, `diagnosticSummary { total: 3, errors: 0, warnings: 3, complete: true }`. `meshTriangleCount` / `assetTriangleCount` 414 both sides of the change; `materialSlots` 7, `materialSlotList` and `unboundSlots: []` identical before and after - which is the point: **not one response field moved while 184 triangles changed material.**

  Disk proof: `SM_WPN_SlotProbe.uasset` 30791 bytes / mtime `2026-09-03 06:35:14 +0300` / md5 `99636c04c0d69de929ea6caaf8c47f89` before, 30611 bytes / `2026-09-05 20:46:05 +0300` / md5 `71d0e696abb4dab8b5893324d94b8bb1` after.

  Read back with `GeometryScript_AssetUtils.copy_mesh_from_static_mesh` + `GeometryScript_Materials.get_triangle_material_id`, part attributed by centroid y (0=a 40=b 80=c 120=d 160=e 200=f 240=g), sub-bucket by centroid x:

```
                              BEFORE (matches this ticket's Measured)   AFTER
part_a  base(x0)              id 0 Alpha      12                       id 0 Alpha      12
part_a  unioned(x20)          id 0 Alpha      12                       id 0 Alpha      12
part_b  base(x0)              id 1 Beta       12                       id 1 Beta       12
part_b  unioned(x20)          id 0 Alpha      12   <-- route 1         id 1 Beta       12   FIXED
part_c  base(x0) walls+caps   id 2 Gamma      48                       id 2 Gamma     112   FIXED
part_c  base(x0) bore wall    id 0 Alpha      64   <-- route 2          (merged into the 112 above)
part_c  unioned(x20)          id 0 Alpha      12   <-- route 1         id 2 Gamma      12   FIXED
part_d  base(x0)              id 3 Delta      32                       id 3 Delta      32
part_d  sibling(x6)           id 3 Delta      10                       id 3 Delta      10
part_e  walls+caps            id 4 Epsilon    48                       id 4 Epsilon   112   FIXED
part_e  bore wall             id 0 Alpha      64   <-- route 2          (merged into the 112 above)
part_f  bevel, no infer       id 0 Alpha      32   <-- route 3         id 5 Zeta       44   FIXED
part_f  bevel, no infer       id 5 Zeta       12                        (merged into the 44 above)
part_g  bevel infer=true      id 6 Eta        44                       id 6 Eta        44   control, unchanged

TOTALS  before  0(Alpha)=208 1(Beta)=12 2(Gamma)=48 3(Delta)=42 4(Epsilon)=48 5(Zeta)=12 6(Eta)=44
        after   0(Alpha)= 24 1(Beta)=24 2(Gamma)=124 3(Delta)=42 4(Epsilon)=112 5(Zeta)=44 6(Eta)=44
```

  `Alpha` falls from 208 of 414 triangles to 24 - exactly `part_a`'s own two boxes, the only legitimate user of slot 0. Every other part is now entirely on its own slot, and no triangle anywhere carries a slot its part did not name. Route 3 was retried first as asked: `part_f` is a part-level `bevel distance=1.0 segments=0` with no `infer_material_id`, and all 44 of its triangles come back on `Zeta`. `part_d` (`self_union`) and `part_g` (`bevel infer_material_id=true`) are byte-identical to before, so the fix did not achieve its result by breaking the controls.

  **The probe stays quiet, as `#5`'s reviewer step 4 requires.** The three warnings are the pre-existing ones - `PWMODEL_UNUNIONED_OVERLAP` on `part_d`'s deliberate overlap, `PWMODEL_UV_OVERLAP_ACROSS_PARTS`, `PWMODEL_FLOATING_COMPONENT` on the deliberate layout. Neither new code fires on a document whose blocks are all single-slot and tagged. `PWMODEL_UNUNIONED_OVERLAP`'s text is the rewritten one: *"Resolve it with 'union { }', which keeps each solid's own material slot - a generator inside the block carries its own material= through the model-wide table"*.

  **Both new diagnostics are reachable**, confirmed with `model.validate` (writes nothing) rather than assumed from the ticket:

  - `PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS`, warning, line-anchored to the op. Case: two overlapping boxes tagged `A` and `B`, `self_union`, then an untagged `subtract`. Message: *"'subtract' produced faces onto geometry carrying 2 different material slots ('A' (0), 'B' (1)), so there is no single slot for them to inherit. They were put on 'A' (0), the slot most of that geometry is on. Write material="<Slot>" on 'subtract' to name the faces it creates, or on a generator inside its block to tag that generator's own surface."*
  - `PWMODEL_BOOLEAN_MATERIAL_UNUSED`, warning. Case per `#6`: a `subtract` block whose first cylinder cuts and whose second, `material="Ghost"`, sits 500 uu away. Message: *"'subtract' opened material slot(s) 'Ghost' (2) that no triangle of the result carries, so the tag did nothing and the asset gains an empty section. A boolean's material= names the faces the operation CREATES, and a tool solid the operation discards produces none - most often one that misses the target while the rest of the block hits it, which is why the op itself did not report doing nothing. Check that solid's placement, drop the tag, or move the geometry it was meant for out of the block."*

  Language surface confirmed against the live parser tables, not the docs: `model.describe_ops { op: "subtract" }` returns `acceptsMaterial: true, boolean: true` with a `material` parameter describing created faces; `model.describe_ops { op: "bevel" }` returns `infer_material_id` **default `true`** and `material_id` defaulting to the dominant slot rather than `0`, plus the new `material=`.

  **One generated wiki page is stale and contradicts the fix.** `Saved/PinWright/wiki/model.describe_ops.md:22` still says *"Boolean ops have `acceptsMaterial: false`: they discard their tool, and `material=` is an error."* - false in all three clauses now, and it is the page an author reads before writing an op. Its source overlay under `Docs/wiki-src/` was missed when `model.authoring.md` was updated. Separately, `PWMODEL_BOOLEAN_MATERIAL_AMBIGUOUS` and `PWMODEL_BOOLEAN_MATERIAL_UNUSED` appear on exactly one wiki page (`model.authoring.materials.md`) and on neither `model.compile.md` nor `model.validate.md`.

  Not verified here, deliberately: `SM_WPN_AR.pwmodel` from its pre-workaround revision (`#5` step 3) - the live rifle and pistol sources are other agents' working files and were not touched. Triangles-per-slot on the `model.compile` response is still absent, as `#5` scoped it; that is why this entry's evidence had to come from a Python read-back rather than from any response field.

  **Recovered on real content, not just on the probe.** Both weapon documents were recompiled under the fixed compiler with their sources unchanged, and the fix returns the `subtract`-opened walls this ticket had recorded as unreachable by any tag:

```
SM_WPN_AR      old compiler                       fixed compiler
  Receiver     17059  75.4%  x[-26.20.. 54.20]    16783  74.1%  x[-26.20.. 33.80]
  Polymer        428   1.9%                         666   2.9%
  Barrel        4751  21.0%                        4819  21.3%
  Optic          384                                384
```

  **`Receiver`'s forward reach drops from x 54.20 to 33.80** — the 157 flash-hider port-wall triangles moving onto `Barrel` where they belong, plus 238 stock bore-wall triangles onto `Polymer`. Those are the walls the ticket's "Workaround, and why it is not a fix" section named as the residue no in-format authoring could reach; they are correct now with the source untouched. The muzzle crown, which is the one surface a player looks at straight down its own axis, was anodised receiver aluminium before this fix and is phosphated barrel after it.
