---
id: B-pwmodel-bevel-perquad-warning-ignores-filter-box
title: "pwmodel `bevel`'s one-group-per-quad warning is measured on the WHOLE mesh and ignores filter_box_min / filter_box_max, so the op's own documented escape hatch still raises the warning it is the escape from"
status: OPEN
severity: Medium
category: bug
tags: [pwmodel, bevel, filter-box, diagnostics, false-positive, model.compile, model.validate]
encounters: 1
lastSeen: 2026-09-03T04:45:00+03:00
---

# The per-quad warning fires identically on the correct and the incorrect spelling

`model.describe_ops { op: "bevel" }` says, verbatim:

> `filter_box_min` / `filter_box_max` are the way out of the second one: they restrict the
> bevel to the edges inside a box.

"The second one" is the one-group-per-quad case. But the warning that names that case is
computed over the whole mesh's polygroup / triangle ratio **before** the filter box is applied,
so taking the documented way out does not clear it. The author is told their bevel will notch
every quad boundary at 3x the triangle cost when the filter box has already prevented exactly
that.

## Repro — two parts, identical geometry, one filtered and one not

Both parts build a bevelled block, append a second block, then bevel again. The only difference
is the filter box on the second `bevel`.

```
part filtered {
    box size=(0.90, 1.04, 0.55) at=(19.45, 0, 3.075) material="Sights" color=(0.230, 0.230, 0.235, 1)
    bevel distance=0.12 segments=0 infer_material_id=true
    union {
        box size=(0.40, 0.48, 0.85) at=(19.50, 0, 3.525) color=(0.230, 0.230, 0.235, 1)
    }
    box size=(1.20, 1.52, 1.15) at=(1.40, 0, 3.375) material="Sights" color=(0.230, 0.230, 0.235, 1)
    bevel distance=0.12 segments=0 filter_box_min=(0.5, -1.0, 2.7) filter_box_max=(2.3, 1.0, 4.1) fully_contained=true
    subtract {
        box size=(1.60, 0.42, 0.60) at=(1.40, 0, 3.80) color=(0.230, 0.230, 0.235, 1)
    }
}

part unfiltered {   # same geometry, shifted +10 in x, second bevel unfiltered
    ...
    bevel distance=0.12 segments=0
    ...
}
```

Result:

| part | second `bevel` | triangles | selfIntersections | warning raised |
|---|---|---|---|---|
| `filtered` | with `filter_box_*` | **130** | **0** | `PWMODEL_STAGE_WARNING` "Mesh has 37 polygroups over 72 triangles" at the bevel's line |
| `unfiltered` | none | **306** (2.35x) | **215** | the same warning, same text, same counts |

The filter box did its job: 130 triangles against 306, and it left the already-chamfered first
block alone instead of chamfering its chamfers, which is where the unfiltered spelling's 215
crossing pairs come from. The warning does not distinguish the two.

## Where it comes from

The message quotes `37 polygroups over 72 triangles`, which is the accumulated mesh at that
moment — the already-bevelled first block plus its union plus the fresh second block. The
filtered bevel only ever touches the second block's own coarse box edges, all of which lie
inside `(0.5, -1.0, 2.7) .. (2.3, 1.0, 4.1)`; the dense region that produced the 37 groups is
at x 19.0..19.9 and is excluded by `fully_contained=true`.

## What it should do

Evaluate the per-quad heuristic over the **edge set the bevel will actually walk** — the
polygroup edges surviving the filter box — rather than over the whole mesh. Failing that,
suppress or qualify it whenever both filter corners are supplied, since the op's own
documentation names the filter box as the remedy for the condition being warned about.

The cost of leaving it is the ordinary cost of a warning that fires on correct code: it trains
authors to skip a message that, unfiltered, is pointing at a genuine 215-crossing defect.

severity rationale: impact=false positive on the op's own documented remedy, on a diagnostic
whose true-positive case is a silently broken surface x reach=any model bevelling more than one
solid in a part -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Hit while bevelling the two sight blocks of
  `Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel` part `sights`, which are appended siblings
  17.45 apart, so the second bevel needed a filter box to avoid re-chamfering the first. A/B in
  one `model.validate` (filtered vs unfiltered, identical geometry shifted 10 in x) gave
  130 tris / 0 crossings vs 306 tris / 215 crossings, and the identical warning on both.
