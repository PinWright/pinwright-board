---
id: B-pwmodel-compound-boolean-tool-oversubdivides-and-leaves-slivers
title: "One `subtract` holding an array_linear of 34 boxes returns 13,106 triangles and 5 zero-area slivers for a solid that needs 950 and 0 — the same cuts issued one boolean at a time are clean"
status: OPEN
severity: High
category: bug
tags: [pwmodel, model-compile, model-validate, boolean, subtract, array_linear, degenerate-triangles, triangle-count, weapons]
encounters: 1
lastSeen: 2026-09-06T06:45:00Z
---

# A compound boolean tool is not the same operation as the cuts it stands for

Siblings inside a boolean block are APPENDED into one tool mesh, which the format documents and
which is fine as a description. What is not fine is the result: the engine retriangulates the whole
affected surface in one pass, and both the triangle count and the number of zero-area triangles in
the output are wildly worse than issuing the same cuts one at a time — for the same solid, to
eleven significant figures.

## What was called

`model.validate` on a Picatinny rail bar, four spellings of one shape. Everything else identical.

```
part rail {
    box size=(37.0, 4.2, 1.0) at=(13.0, 0, 8.5) material="Receiver"
    bevel distance=0.12 segments=0 infer_material_id=true
    subtract material="Receiver" {
        box size=(0.58, 5.0, 0.55) at=(-4.8, 0, 8.85) material="Receiver"
        array_linear count=N offset=(1.083, 0, 0)
    }
}
```

against the same 34 boxes written as 34 separate one-box `subtract` blocks at
`x = -4.8 + 1.083k`.

## What happened — measured, `model.validate`, UE 5.8

| spelling | meshTriangleCount | degenerateTriangles | isClosed | signedVolume |
|---|---|---|---|---|
| one tool of 34 boxes | 13,106 | **5** | true | 119.27761905788446 |
| one tool of 17 boxes | 4,008 | **11** | true | 136.7357340030941 (17 slots) |
| one tool of 1 box | 68 | 0 | true | 153.16690101034976 (1 slot) |
| **34 tools of 1 box each** | **950** | **0** | true | **119.27761905788527** |

The first and last rows are the SAME SOLID — same bounds, same signed volume to eleven figures —
and one of them carries **13.8x the triangles** and five zero-area triangles the other does not.

**The sliver count is not a placement artefact.** It went UP when the tool lost half its boxes
(34 -> 5, 17 -> 11, 1 -> 0), and no operand nudge moved it toward zero: tool y-overrun 5.0 / 5.2 /
5.6 / 7.0 gave 5 / 7 / 3 / 60 slivers, and `bevel distance` 0.10 instead of 0.12 gave 3. Isolated,
neither ingredient produces one: the bevelled bar alone is 44 tri / 0 degenerate, and the same 34
slots cut into an UNbevelled bar are 5,110 tri / 0 degenerate. It is the combination plus the
compound tool.

## Why it matters beyond the triangles

Those 5 slivers survive into the static mesh's **source model** (the build's `bRemoveDegenerates`
only cleans render data), and `geometry.audit_static_meshes` reads the source model at the default
`lodType: MaxAvailable`. A degenerate component is UNKNOWN, and one unknown component makes the
whole asset's `inverted` check **unrunnable** — so `SM_WPN_AR` had no orientation verdict at all
for three review rounds. Two more slivers on the same asset came from a different cause
(a filtered `bevel` reaching buried edges); with both fixed at source the audit now answers
`inverted: clean` on all four weapon meshes.

There is also no way out downstream — see
`B-pwmodel-remove-degenerates-deletes-instead-of-repairing-and-opens-mesh`.

## What was expected

That `subtract { tool; array_linear count=34 }` produces the same mesh as 34 `subtract`s of the
same boxes, or that the difference is documented. The format sells the array form as the
maintainable spelling ("the pitch is one number to edit") and it is the one every example uses.

## Workaround

Write the boolean out per instance. `Content/FPS/Weapons/Meshes/SM_WPN_AR.pwmodel`, `part rail`,
now carries 34 one-box `subtract` lines and a note saying not to re-fold them.

## Root cause — guess, no source read taken

The appended tool mesh is one shell with 34 disjoint lobes; the boolean's cut-loop insertion and
retriangulation appear to run over the whole intersected polygroup rather than per lobe, so every
slot's cut participates in one large planar retriangulation of the bar's top face and its chamfer
strips. That would explain both symptoms at once — the triangle blow-up and the near-collinear
slivers — and it predicts the chaotic dependence on operand count. **Inference from the
measurements; no plugin source was opened for this ticket.**

## Severity

**High.** It is silent (a clean `success: true`, closed mesh, right volume, right bounds), it
multiplies triangles by an order of magnitude on the exact construction the format recommends, and
its degenerate output disables an entire audit check on the asset with no in-format remedy.

## Related

- `B-pwmodel-remove-degenerates-deletes-instead-of-repairing-and-opens-mesh` — the reason the
  slivers cannot be cleaned after the fact.
- `B-pwmodel-bevel-after-cut-arris-self-intersects` — why the bevel has to run BEFORE the cuts here,
  which is what puts a chamfer strip under the tool in the first place.
- `E-geometry-array-radial-merges-in-place`, `B-array-linear-first-copy-at-origin` — the array ops.

## History
- `#1-filed` `OPEN` reporter — Filed while closing WEAPONS review 04 defect 3 (7 degenerate
  triangles on `SM_WPN_AR`). Four spellings measured with `model.validate` on the isolated rail
  part; table above. Fixed at source by writing 34 one-box `subtract` blocks, which took the whole
  model from 23,266 to 11,110 mesh triangles and from 7 to 0 degenerates, and made
  `geometry.audit_static_meshes` `inverted` runnable and clean on all four weapon meshes for the
  first time.
