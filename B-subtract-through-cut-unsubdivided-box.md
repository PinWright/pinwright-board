---
id: B-subtract-through-cut-unsubdivided-box
title: "`.pwmodel` `subtract` refuses a through-cut on a default-tessellation `box` and aborts with `PWMODEL_BOOLEAN_NO_EFFECT`, whose message asserts the operands do not intersect when they overlap by 40% of the target's volume"
status: OPEN
severity: High
category: bug
tags: [pwmodel, subtract, boolean, tessellation, through-cut, false-diagnostic, PWMODEL_BOOLEAN_NO_EFFECT]
encounters: 1
lastSeen: 2026-08-27T18:55:42+05:00
---

# `subtract` will not cut clean through an unsubdivided `box`, and blames the operands for not intersecting

A `.pwmodel` `subtract` whose tool slices completely across a default-tessellation
`box` (12 triangles, 8 corner vertices) fails the compile with:

```
PWMODEL_BOOLEAN_NO_EFFECT: 'subtract' changed nothing: the block's geometry does not
intersect the geometry accumulated so far. Move or resize the block's geometry so it
overlaps, or delete the op ...
```

**The operands overlap by 40% of the target's volume.** The sentence is not
merely unhelpful, it is false, and it sends the author off to move geometry that
is already in the right place.

## Verbatim repro

A 100-cube and a slab that takes its top off. Five `model.validate` calls, no
files, argument values exactly as run:

| document | result |
|---|---|
| `box size=(100,100,100)` + `subtract { box size=(300,300,100) at=(0,0,50) }` | **works** — 16 tris, z -50..0 |
| same, tool `at=(0,0,60)` | **BOOLEAN_NO_EFFECT** |
| same, tool `at=(0,0,70)` | **BOOLEAN_NO_EFFECT** |
| same, tool `box size=(300,300,300) at=(0,0,190)` | **BOOLEAN_NO_EFFECT** |
| same, tool `box size=(300,100,100) at=(0,0,60)` | **BOOLEAN_NO_EFFECT** |

**Two controls isolate it to the TARGET's tessellation, not to the overlap:**

| document | result |
|---|---|
| target `box size=(100,100,100) segments=(3,3,3)` + tool `at=(0,0,60)` | **works** — 60 tris, z -50..10, `signedVolume` 600000 |
| target unsubdivided + tool `box size=(300,300,100) segments=(4,4,4) at=(0,0,60)` | **BOOLEAN_NO_EFFECT** |

**And one control that shows a through-cut is the specific shape at fault** — a
tool that stays inside the target's own footprint cuts an unsubdivided box
happily at the same depth:

| document | result |
|---|---|
| `box size=(100,100,100)` + `subtract { box size=(40,40,40) at=(0,0,50) }` | **works** — 32 tris, `signedVolume` 968000 = 1000000 - 40x40x20 |

So, in one line each:

- **Subdividing the TARGET fixes it.**
- **Subdividing the TOOL does not.**
- **A notch works; a through-cut does not.**
- **The one through-cut that does work is the one whose cut plane lands exactly
  on z = 0.**

`signedVolume` on the working cases — **600000** for the 100x100x60 remainder and
**968000** for the notch — confirms the geometry is right whenever the op is
allowed to run at all, so **nothing is wrong with the operands**.

## Root cause (guilty source line)

**Two halves, and only one is traced.**

**The message half is source-confirmed.** The false sentence is emitted at
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

The check is `!OpResult.bChanged` — "the engine returned the target unmodified".
The **message** translates that into "the operands do not intersect", which is a
strictly stronger claim than the condition tested and is measurably untrue here.
Nothing in it points at tessellation.

**The geometry half is NOT traced.** Why the engine boolean declines a
through-cut on a 12-triangle box while accepting the same cut on a 60-triangle
one, and why the z = 0 cut plane is the one exception, is **inference from the
response readings tabulated above**, not from reading the engine's boolean
implementation. A fixer should treat that as the open question, not as an
established mechanism.

## What it should do

Two parts, both worth doing.

1. **Find out why the engine boolean declines a through-cut on a 12-triangle
   box**, and either pre-subdivide the target on this path or route it through
   the path that works. This is the actual capability gap; the diagnostic is
   downstream of it.
2. **Failing that, change the diagnostic at `PwModelCompiler.cpp:2502-2506`.** It
   should not claim the operands are disjoint unless their bounding boxes
   actually are — both meshes are in hand at that point (`Mesh` and `Tool`, see
   `E-boolean-no-effect-names-no-bounds` for the lifetime caveat on reading
   `Tool` there). When they **do** overlap and the boolean still returned the
   target unchanged, that is a different fact and deserves a different sentence —
   one that names tessellation and the `segments=` remedy.

## Workaround

Give the **TARGET** box a `segments=` above 2 on the axes the cut crosses;
`segments=(3,3,3)` is enough. Every box in `SM_Temple_Podium`,
`SM_Temple_Cella` and `SM_Temple_Entablature` on this build carries a `segments=`
for this reason as well as for erosion noise.

## Distinct from related tickets

- **Do NOT fold this into `B-boolean-subtract-ignores-tool-offset`** (IN-REVIEW,
  High). Three separate reasons, any one of which is sufficient:
  1. **Different namespace.** That ticket is the `geometry.*` RPC family
     (`geometry.create_box`, `geometry.boolean_subtract` on spawned actors); this
     is the `.pwmodel` `subtract` op inside `FCompiler::RunBoolean`.
  2. **Different cause.** Its root cause is a spawn transform **double-applied**
     — the same `FTransform` fed to both the `Append*` primitive call and
     `SpawnDynamicMeshActor` — cited to `PrimitiveHandler.cpp:104-127`. Nothing
     here is placed by an actor transform: `.pwmodel` bakes every op's
     `at`/`rotate`/`scale` into vertices and passes identity transforms to the
     boolean (`PwModelCompiler.cpp:2466-2467`).
  3. **Different operand geometry.** In its repro the operands **genuinely ARE
     disjoint** — the cutter's world footprint has been shifted to 2x its
     requested offset and no longer touches the target, so the engine correctly
     finds zero intersection. Here the operands demonstrably **do** overlap, by
     40% of the target's volume, and the boolean still declines. Its failure is
     also a *silent* `success:true`; this one is a loud abort.
- `E-boolean-no-effect-names-no-bounds` (OPEN, Medium) is the sibling filed from
  the same session against the **same emit site**: that ticket is about the
  message naming no position for either solid, this one is about the message
  asserting a fact that is false. Fix them together while
  `PwModelCompiler.cpp:2500-2507` is open — but they are separately actionable,
  and (1) above is not in either of them.
- A read-only sweep of the whole board this session found **no** other ticket on
  the `.pwmodel` `subtract` / boolean path.

severity rationale: impact=the caller is told a measurably false fact and acts on it — the diagnostic asserts the operands are disjoint when they overlap by 40% of the target's volume, and names nothing that would lead to the real remedy, so the author moves correct geometry indefinitely; partially offset by the abort being loud rather than silent and by a one-parameter workaround existing x reach="take the top off a wall / beam / step with a big slab" is the single most common boolean in architectural authoring and a plain `box` is the first target anyone reaches for -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis temple ruins (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. All rows measured live through `model.validate`, values copied from the responses. Five tool-position rows on a `box size=(100,100,100)` target: tool `box size=(300,300,100) at=(0,0,50)` **works** (16 tris, z -50..0); the same tool at `at=(0,0,60)` and `at=(0,0,70)`, a `size=(300,300,300) at=(0,0,190)` and a `size=(300,100,100) at=(0,0,60)` all abort with `PWMODEL_BOOLEAN_NO_EFFECT`. Two tessellation controls: target `segments=(3,3,3)` + tool `at=(0,0,60)` **works** (60 tris, z -50..10, `signedVolume` 600000), while giving the **tool** `segments=(4,4,4)` against an unsubdivided target still fails. Notch control: `subtract { box size=(40,40,40) at=(0,0,50) }` on an unsubdivided target **works** (32 tris, `signedVolume` 968000 = 1000000 - 40x40x20). Net: subdividing the TARGET fixes it, subdividing the TOOL does not, a notch works, a through-cut does not, and the one through-cut that works is the one whose cut plane lands exactly on z = 0. The working `signedVolume` figures confirm the geometry is correct whenever the op runs, so the operands are sound. Message half source-confirmed at HEAD in this tree: emitted at `PwModelCompiler.cpp:2500-2507` in `FCompiler::RunBoolean` (`:2416`), constant at `PwModelDiagnostic.h:182`; the tested condition is `!OpResult.bChanged` while the sentence claims the operands do not intersect — strictly stronger than what was checked, and false here. Geometry half NOT traced into engine source: why a 12-triangle box refuses a through-cut a 60-triangle one accepts, and why z = 0 is the exception, is stated as inference from the tabulated responses. Worked around by carrying `segments=` on every boolean target box in `SM_Temple_Podium` / `SM_Temple_Cella` / `SM_Temple_Entablature`; defect untouched.
