---
id: E-boolean-no-effect-names-no-bounds
title: "`PWMODEL_BOOLEAN_NO_EFFECT` tells the author to move the cutter but names no position for either solid — the failing part's bounds are dropped and the tool mesh never becomes anything measurable"
status: OPEN
severity: Medium
category: ergonomic
tags: [pwmodel, subtract, boolean, diagnostic, bounds, authoring-cost, PWMODEL_BOOLEAN_NO_EFFECT]
encounters: 1
lastSeen: 2026-08-27T18:55:42+05:00
---

# `PWMODEL_BOOLEAN_NO_EFFECT` says "move it" without saying where either body is

The message is:

```
'subtract' changed nothing: the block's geometry does not intersect the geometry
accumulated so far. Move or resize the block's geometry so it overlaps, or delete the op -
a boolean that removes nothing leaves the model without the feature it describes.
```

It names the op and the line, and then asks the author to move the cutter
**without saying where either body is**.

- The response carries **no bounds for the failed part** — the whole part is
  aborted, so `parts[]` holds only the parts that already succeeded.
- It carries **none at all for the tool mesh**, which never becomes geometry an
  author can measure.
- The one number that would settle it — how far apart they are — is exactly what
  `floatingGeometry.nearestDistance` already computes for a different check.

**The abort itself is correct** and this ticket does not ask for it to change:
`model.validate`'s own notes record twelve green examples of visibly broken
models that shipped when this was a warning. The defect is that **the remedy it
names cannot be acted on from the information it gives**.

## Verbatim repro

On `Content/Atlantis/Meshes/SM_Statue_Torso.pwmodel`, a fracture cutter placed by
plane arithmetic:

```
part body {
    cone base_radius=228 top_radius=200 segments=8 from=(0, 0, 190) to=(12, 4, 600)
    cone base_radius=202 top_radius=178 segments=8 from=(10, 4, 580) to=(0, -12, 940)
    cone base_radius=180 top_radius=236 segments=8 from=(2, -10, 920) to=(-14, 0, 1460)
    sphere radius=190 subdivisions=4 scale=(1.0, 0.72, 1.05) at=(-6, 0, 1220)
    noise_deform magnitude=13 frequency=0.008 seed=2525
    noise_deform magnitude=5 frequency=0.03 seed=3525
    subtract { box size=(1600, 1600, 1600) at=(0, 274, 2152) rotate=(-20, 0, 0) }
    subtract { box size=(700, 700, 700) at=(0, -460, 1520) rotate=(20, 8, 0) }
}
```

The second `subtract` returns `PWMODEL_BOOLEAN_NO_EFFECT`. **Nothing in the
response says that the first cut had already taken the body's top down to
z = 1277.** Finding that out required deleting the second op and re-running
`model.validate` on the remainder — a whole extra compile cycle per attempt. It
took three.

**The plumbing demonstrably exists:** the successful-part bounds ARE present in
the same response, for the sibling `stump` part. It is the *failing* part and the
tool that have none.

## Root cause (guilty source line)

Source-confirmed at HEAD in this tree. The emit site is
`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:2500-2507`,
inside `FCompiler::RunBoolean` (opens `:2416`); the code constant is declared at
`Model/PwModelDiagnostic.h:182`:

```cpp
    if (!OpResult.bChanged)
    {
        Error(PwModelDiagnosticCodes::PWMODEL_BOOLEAN_NO_EFFECT, Op,
            FString::Printf(TEXT("'%s' changed nothing: the block's geometry does not intersect the geometry accumulated so far. ")
                            TEXT("Move or resize the block's geometry so it overlaps, or delete the op - a boolean that removes ")
                            TEXT("nothing leaves the model without the feature it describes."),
                *Op.OpName));
        return false;
    }
```

`*Op.OpName` is the **only** value formatted into the message. Both operands are
named locals at that point — the accumulated part is the `UDynamicMesh* Mesh`
parameter (`:2416`) and the tool is `TStrongObjectPtr<UDynamicMesh> Tool`
(`:2421`) — so the data a useful message needs is in scope.

## Lifetime caveat that changes the fix — read this before writing code

**`Tool->MarkAsGarbage()` is called at `PwModelCompiler.cpp:2470`, thirty lines
before the emit at `:2500-2507`.**

```cpp
2466:        OpResult = GeometryOps::Boolean(
2467:            Mesh, FTransform::Identity, Tool.Get(), FTransform::Identity, Operation, Params);
2468:    }
2469:
2470:    Tool->MarkAsGarbage();
```

The `TStrongObjectPtr` still holds the pointer, so a same-frame bounds read at
the emit site would not crash today. But it is a garbage-marked object, and
**any fix that reads tool bounds at the emit site must either move the read above
`:2470`, or move `MarkAsGarbage` below the diagnostic.** Capturing the tool's
bounds into a plain `FBox` local before `:2470` is the cheaper of the two and
does not reorder object lifetime at all.

## What it should do

Put **two boxes** in the diagnostic — the accumulated part's bounds and the
tool's bounds — plus their separation. Both are in hand at the moment the check
fires (subject to the caveat above), and between them they turn *"move it"* into
*"move it 47 units in -Z"*.

## The compounding second gap

`rotate=` on a cutter moves the cutter in the **same axis the overlap depends
on**. A box placed by "point plus half-size along the rotated normal" is only
right if the author guessed the sign convention of `roll` in `FRotator`'s local
+Z — and **there is no measurement anywhere in the response to check that guess
against**. Two of the three failed attempts on `SM_Statue_Torso` were sign
errors that were invisible until the part was rebuilt without the op.

This is why bounds in the diagnostic are worth more here than they would be for a
purely translational miss: the author is not just misplaced, they cannot tell
which of two hypotheses they are wrong about.

## Workaround

Place cutters by **OVERLAP, not by plane arithmetic**: size the cutter so its
*unrotated* extent already straddles the target, and confine rotation to the axis
the overlap does not depend on (a `yaw` for a cut whose overlap is in z). Written
into `SM_Statue_Torso.pwmodel`'s header. To find where the accumulated geometry
actually is, delete the failing op and `model.validate` the remainder — the
bounds come back on the part that then succeeds.

## Distinct from related tickets

- `B-subtract-through-cut-unsubdivided-box` (OPEN, High) is the sibling filed
  from the same session against the **same emit site**, and the two must not be
  merged: that one is a **correctness** defect — the boolean declines a
  through-cut on a 12-triangle target that it accepts on a 60-triangle one, and
  the message asserts the operands are disjoint when they overlap by 40% of the
  target's volume. This one is **authoring cost** — the abort is correct and the
  operands really do miss; the message simply cannot be acted on. Fix them
  together while `PwModelCompiler.cpp:2500-2507` is open; they touch the same
  lines and neither subsumes the other.
- `B-boolean-subtract-ignores-tool-offset` (IN-REVIEW, High) is the `geometry.*`
  RPC family, caused by a spawn transform double-applied at
  `PrimitiveHandler.cpp:104-127`, and its failure is a *silent* `success:true`.
  Different namespace, different cause, opposite loudness.
- A read-only sweep of the whole board this session found **no** other ticket on
  the `.pwmodel` `subtract` / boolean path, and `PWMODEL_BOOLEAN_NO_EFFECT`
  appeared nowhere on it before these two.

severity rationale: impact=soft blocker — the abort is correct and the model is recoverable, but the only remedy the message names is unactionable, so each attempt costs a full delete-op-and-revalidate compile cycle (three of them on the case measured), compounded by `rotate=` moving the cutter in the overlap axis with no measurement in the response to check the sign convention against x reach=fires on the most common boolean shape in architectural authoring, and the missing data is already computed for a sibling check (`floatingGeometry.nearestDistance`) and already reported for sibling parts -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Found building `Content/Atlantis/Meshes/SM_Statue_Torso.pwmodel` on the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Second `subtract { box size=(700,700,700) at=(0,-460,1520) rotate=(20,8,0) }` returned `PWMODEL_BOOLEAN_NO_EFFECT`; the response named neither the accumulated part's bounds (the part is aborted, so `parts[]` holds only the parts that already succeeded) nor the tool's (it never becomes measurable geometry). The fact that settled it — the first cut had already taken the body's top down to z = 1277 — was only recoverable by deleting the op and re-running `model.validate` on the remainder, one extra compile cycle per attempt, three attempts. Plumbing proven present: the sibling `stump` part's bounds ARE in the same response. Source-confirmed at HEAD in this tree: emitted at `PwModelCompiler.cpp:2500-2507` inside `FCompiler::RunBoolean` (`:2416`), constant declared at `PwModelDiagnostic.h:182`, and `*Op.OpName` is the only value formatted in — while both operands are named locals (`UDynamicMesh* Mesh` parameter, `TStrongObjectPtr<UDynamicMesh> Tool` at `:2421`). **Lifetime caveat recorded for the fixer: `Tool->MarkAsGarbage()` runs at `:2470`, thirty lines before the emit** — the `TStrongObjectPtr` keeps the pointer alive so a same-frame read would not crash, but any fix reading tool bounds at the emit site must move the read above `:2470` or move `MarkAsGarbage` below the diagnostic; capturing an `FBox` local before `:2470` is cheapest. Compounding gap also recorded: `rotate=` moves a cutter in the same axis the overlap depends on, so "point plus half-size along the rotated normal" is only right if the author guessed the sign convention of `roll` in `FRotator`'s local +Z, and no measurement in the response checks that guess — two of the three failures were sign errors. Classified ergonomic, not correctness: the abort is right (see `model.validate`'s note on twelve green examples of visibly broken models). Worked around by placing cutters by overlap rather than plane arithmetic, written into the `.pwmodel` header; defect untouched.
