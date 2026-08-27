---
id: E-pwmodel-transform-rotate-pivots-origin
title: "`transform rotate=` pivots about the part-local ORIGIN, so tilting a boolean tool authored at z=1790 by 7 degrees also translates it 218 uu sideways — undocumented, unwarned, and the whole health gate stays green while 168 triangles hang 140 uu clear of the model"
status: OPEN
severity: Medium
category: ergonomic
tags: [pwmodel, transform, rotate, boolean, subtract, lever-arm, part-local-space, floatingGeometry, docs-gap, false-green]
encounters: 1
lastSeen: 2026-08-27T18:57:03+05:00
---

# Tilting a tool moves it, and only `floatingGeometry` notices

Tilting a boolean tool with a `transform rotate=` op moves the tool **sideways** as well as turning
it, because the rotation pivots about the **part-local origin** rather than about the accumulated
geometry. The displacement is a lever arm:

```
displacement = distance_from_origin * sin(angle)
```

A tool authored at `z = 1790` and tilted 7 degrees is therefore displaced `1790 * sin(7deg) =
218 uu` in X. It cuts the wrong thing, and the compile is green.

The behaviour is arguably **correct**: `model.authoring` § *Transform spaces* does say the
`transform` op acts *"in part-local space"*. But the page never says the **rotation pivots about the
origin**, and an author's mental model of "tilt the tool" is rotation in place. Filed as `E-`
accordingly — documentation plus an optional warning, not a behaviour change.

## Root cause (guilty source lines — verified, and it is the documented behaviour)

`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:2401-2406`:

```cpp
    if (Name == TEXT("transform"))
    {
        UGeometryScriptLibrary_MeshTransformFunctions::TransformMesh(
            Mesh, ReadTransformParams(P), /*bFixOrientationForNegativeScale=*/true, nullptr);
```

and `ReadTransformParams` (`Source/PinWright/Private/PwSource/PwValueRead.h:276-281`):

```cpp
        const FVector At    = GetVector3(Params, TEXT("at"),    FVector::ZeroVector);
        const FVector Scale = GetVector3(Params, TEXT("scale"), FVector::OneVector);
        return FTransform(GetRotator(Params, TEXT("rotate")), At, Scale);
```

An `FTransform` applies **scale, then rotation, then translation**, all about the origin, and
`TransformMesh` applies it to every vertex position. With no `at=`, `At` is `(0,0,0)`, so the
rotation is about `(0,0,0)` and there is nothing between the geometry's real position and the pivot.
`scale=` has the same lever arm for the same reason.

Nothing here is a wrong line — it is what the op says it does. The defect is that the format's own
page never states the pivot, and no warning fires when the pivot is far from the geometry.

## Verbatim repro

`SM_Column_Broken_A.pwmodel`, a break tool over a shaft of **radius 155**. Measured 2026-08-27,
UE 5.8, this checkout:

```
subtract {
    box size=(700, 700, 700) segments=(10, 10, 4) at=(0, 0, 1790)
    noise_deform magnitude=95 frequency=0.0055 seed=2201
    transform rotate=(0, 7, 0)          # <- pivots about (0,0,0), not about the box
}
```

`1790 * sin(7deg) = 218.2`. The box's side wall lands at **x = -132**, inside the radius-155 shaft,
and slices a crescent off one flank instead of taking the top off.

The correct in-format spelling is the generator's own `rotate=`, which composes into the primitive
**before** its `at=`:

```
    box size=(700, 700, 700) segments=(10, 10, 4) rotate=(0, 7, 0) at=(0, 0, 1790)
```

That turns the box about itself and lands it where it was asked for.

## The whole health gate is green on the broken spelling

The broken spelling compiled:

```
success: true          isClosed: true            signedVolume: positive
boundaryEdges: 0       degenerateTriangles: 0    nonManifoldVertices: 0
```

— the entire documented health gate — while leaving **168 triangles detached and hanging 140 uu
clear of the column**.

The **only** field that reported it was `floatingGeometry`:

```
floatingCount: 1     nearestDistance: 140.83     center: (118.4, 11.1, 1697.2)
```

An earlier variant of the same mistake produced `floatingCount: 2` (20 and 28 triangles).

That signal is luck, not coverage: it fires because this particular miss **detached** something. A
tool that is displaced onto a different part of the same solid, or off the model entirely, removes
the wrong material or nothing at all and leaves **no** signal in the response.

## Impact and reach

Every break face, every collapsed roof and every chipped edge on this build is a tool authored far
from the origin — the trap is on the main path for ruin geometry, and the lever arm grows with the
height at which the tool is placed, so the taller the model the worse it gets. It is also the second
time in one session that `floatingGeometry` was the sole signal for a boolean gone wrong; the other
is `B-pwmodel-sphere-subdivisions-extent`, filed in this same run.

## What it should do

- **Documentation, in `model.authoring` § Transform spaces** (the fix that matters): state that
  `transform rotate=` and `transform scale=` **pivot about the part-local origin**, that a lever arm
  of `distance * sin(angle)` follows, and name the generator's own `rotate=` as the in-place
  alternative. The same sentence belongs on the `transform` op's `describe_ops` text
  (`Model/PwModelParser.cpp:900`).
- **Optionally, a warning**: when a `transform rotate=` displaces the accumulated geometry's centre
  by more than some multiple of the geometry's own size, that is the case nobody means.

## Workaround

Put `rotate=` on the tool's own generator, never a `transform rotate=` op, whenever the tool is
authored away from the part-local origin. Applied to both broken columns here.

## Distinct from related tickets

- **`E-warp-deformers-no-axis-or-center`** (OPEN, Medium) is a **partial overlap** — same premise,
  different op, different failure. It is about `GeometryOps::Bend` / `Twist` / `Taper` passing
  `FTransform::Identity` as the engine's gizmo frame, so those three ops only work on geometry
  aligned to part-local Z and straddling the part-local origin: a **failure to REACH** off-origin
  geometry. This ticket is `transform rotate=` silently **DISPLACING** off-origin geometry by
  `distance * sin(angle)`. The shared premise — a `.pwmodel` op silently taking the part-local origin
  as its frame, undocumented and unwarned — is real and worth fixing as one theme, but neither fix
  resolves the other.
  **This ticket also invalidates that ticket's prescribed workaround.** It tells the author to
  "author the shape at the origin along +Z, warp it, then `transform at= rotate=` it into place" —
  and `transform rotate=` on geometry that is no longer at the origin is precisely this trap. That
  workaround is safe only under an unstated precondition: **the shape must still be at the origin
  when the `transform` runs**, which fails the moment the shape has an `at=` before it. An encounter
  section recording that precondition has been appended there; its existing text is untouched.
- No other board ticket mentions `floatingGeometry`, `floatingCount` or `nearestDistance` — a
  read-only sweep of all 1316 confirmed zero occurrences board-wide.

severity rationale: impact=soft blocker — the op behaves exactly as its own "part-local space" contract says and a correct in-format spelling exists (`rotate=` on the generator), so this is a documentation gap rather than wrong behaviour, and it is Medium rather than Low because the consequence is silently wrong geometry on a compile that passes the entire documented health gate, caught here only by chance (`floatingGeometry` fires only when the miss happens to DETACH something; a tool displaced onto another part of the same solid leaves no signal at all) × reach=every break face, collapsed roof and chipped edge on this build is a tool authored far from the origin, and the lever arm grows with height — the main path for ruin geometry, though not every document -> Medium. Not High: no response field returns wrong or stale data; every value is true of the mesh that was actually built.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD, on `SM_Column_Broken_A.pwmodel`. `subtract { box size=(700,700,700) segments=(10,10,4) at=(0,0,1790); noise_deform magnitude=95 frequency=0.0055 seed=2201; transform rotate=(0,7,0) }` over a radius-155 shaft: the `transform` pivots about the part-local origin, so `1790 * sin(7deg) = 218` uu of displacement in X puts the box's side wall at **x = -132**, inside the shaft, slicing a crescent off one flank instead of taking the top off. The generator's own `rotate=`, which composes into the primitive BEFORE its `at=` (`box ... rotate=(0,7,0) at=(0,0,1790)`), turns the box about itself and is the correct in-place spelling. The broken spelling compiled `success: true`, `isClosed: true`, `signedVolume` positive, `boundaryEdges: 0`, `degenerateTriangles: 0`, `nonManifoldVertices: 0` — the ENTIRE documented health gate green — while leaving **168 triangles detached and hanging 140 uu clear of the column**. `floatingGeometry` was the SOLE signal: `floatingCount: 1`, `nearestDistance: 140.83`, `center: (118.4, 11.1, 1697.2)`; an earlier variant gave `floatingCount: 2` (20 and 28 triangles). That signal is luck rather than coverage — it fires only because this miss detached something; a tool displaced onto another part of the same solid leaves nothing in the response. Source-verified at HEAD: `PwModelCompiler.cpp:2401-2406` calls `TransformMesh(Mesh, ReadTransformParams(P), ...)` and `ReadTransformParams` (`PwValueRead.h:276-281`) returns `FTransform(GetRotator(rotate), At, Scale)` — an FTransform applies scale, then rotation, then translation, all about the origin, and with no `at=` the pivot is `(0,0,0)`; `scale=` carries the same lever arm. Nothing here is a wrong line: the behaviour matches the op's own contract. Classified `E-` because the defect is that `model.authoring` § Transform spaces says `transform` acts "in part-local space" but never says the rotation PIVOTS about the origin, and nothing warns. Fix: state the pivot and the `distance * sin(angle)` lever arm in `model.authoring` § Transform spaces and on the `transform` op text (`PwModelParser.cpp:900`), naming the generator's `rotate=` as the in-place alternative; optionally warn when a `transform rotate=` displaces the accumulated geometry's centre by more than some multiple of its own size. Reach: every break face, collapsed roof and chipped edge on this build is a tool authored far from the origin, and the lever arm grows with the height of the tool. Worked around by moving `rotate=` onto the generator on both broken columns; defect untouched. Deduped: NEW — partial overlap with `E-warp-deformers-no-axis-or-center` (shared premise: an op silently taking the part-local origin as its frame) but a different op and the opposite failure (failing to REACH off-origin geometry vs. DISPLACING it); an encounter has been appended there recording that this evidence contradicts that ticket's prescribed workaround, which is safe only while the shape is still at the origin when the `transform` runs. `floatingGeometry` / `floatingCount` / `nearestDistance` have zero occurrences board-wide across all 1316 tickets.
