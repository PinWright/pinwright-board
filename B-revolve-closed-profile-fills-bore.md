---
id: B-revolve-closed-profile-fills-bore
title: "`revolve angle=360` with a closed off-axis profile caps to the AXIS and fills the bore with a membrane — `isClosed`, `boundaryEdges`, `orientationConsistent` and even `signedVolume` all read clean"
status: DONE
severity: High
category: bug
tags: [pwmodel, revolve, capped, silent-wrong-geometry, health-gate, bore, axis-cap]
encounters: 2
lastSeen: 2026-08-28T08:30:00+05:00
---

# `revolve` fills the bore of a ring and the whole health gate passes — a full revolution of a closed off-axis profile emits two axis caps that cancel exactly in `signedVolume`

A `revolve` whose profile is a **closed section loop** — last point repeating the
first, none of the points on the axis — at `angle=360` with `capped` left at its
default `true` produces the annulus you asked for **plus a flat membrane across
the bore**. Nothing in the response says so. This is the documented way to author
a ring, a torus of non-circular section, a tube with a shaped wall, a wheel rim,
a pipe flange: everything whose section is a closed loop off the axis. Every one
of them comes out with its hole filled.

Measured on `/Game/Atlantis/Meshes/SM_Portal_Ring`:

| field | value | what it should have said |
|---|---|---|
| `success` | true | — |
| `isClosed` | true | true |
| `boundaryEdges` | 0 | 0 |
| `orientationConsistent` | true | true |
| `degenerateTriangles` | 0 | 0 |
| `signedVolume` | 34,184,797 | 34,184,797 — **the correct annulus volume** |
| `meshTriangleCount` | 648 | 576 |
| diagnostics | none | — |

`model.compile`'s own page publishes the gate `health.isClosed &&
health.signedVolume > 0` (`Saved/PinWright/wiki/model.compile.md:57`). **That gate
passes on the broken mesh.** On this build it would have shipped a portal you
cannot see through, in the level's centrepiece shot.

## Why `signedVolume` stays correct

The two axis caps are **coincident and oppositely wound**, so their contributions
to the signed volume cancel *exactly*. The membrane adds surface, not enclosed
volume, so the one number the published gate reads is the one number the defect
cannot move.

The only tell in the numbers is the triangle count: **648 against the 576 the
profile arithmetic predicts** — 8 profile edges x 36 steps x 2 triangles per quad
= 576. The 72-triangle excess is exactly two 36-triangle fans, one per axis cap.
Nobody computes that by hand, so in practice **the defect is only visible in a
render**.

## Root cause (guilty source line)

**The geometry half is inference from measured responses; it was not traced into
the compiler.** What the responses establish is that at `angle=360` on a profile
whose endpoint is repeated, the op emits two triangle fans to the axis rather
than welding the profile's coincident seam to itself.

**The documentation half has a real line.**
`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:556-562`:

```cpp
    Ops.Add(MakeGenerator(TEXT("revolve"), TEXT("A surface of revolution swept from a 2D profile."), {
        MakeRequired(TEXT("profile"), EPwModelParamType::PointList2,
            TEXT("Profile points [(x, y), …] revolved around the local Z axis.")),
        MakeParam(TEXT("angle"), EPwModelParamType::Number, TEXT("360"), TEXT("Revolution sweep in degrees.")),
        MakeParam(TEXT("steps"), EPwModelParamType::Integer, TEXT("16"), TEXT("Revolution steps. Clamped to 2-256 …")),
        MakeParam(TEXT("capped"), EPwModelParamType::Bool, TEXT("true"), TEXT("Cap the ends of a partial revolution.")),
    }));
```

`:561` is the guilty text. **"Cap the ends of a partial revolution"** reads as
"only relevant when `angle` < 360", which is exactly why the flag was left at its
default on a full revolution. In fact it controls the caps **to the AXIS**, which
are precisely what a full revolution of an off-axis profile needs suppressed.
`model.describe_ops {op:"revolve"}` serves this string verbatim, so the author's
first and best source of truth actively points away from the fix.

## Verbatim repro

Two `model.validate` calls, inline text, no file needed.

```
part a {
    revolve profile=[(655, -33), (667, -45), (733, -45), (745, -33), (745, 33), (733, 45), (667, 45), (655, 33), (655, -33)] steps=36
}
```
-> 648 triangles, `isClosed` true, `boundaryEdges` 0, `signedVolume` 34,184,797,
no diagnostics. **Bore filled.** Rendered at
`Saved/Screenshots/AssetPreview/atl_portal_ring_q.png`: a solid disc with a
six-wedge fan pattern where the opening should be.

```
part a {
    revolve profile=[...same nine points...] steps=36 capped=false
    weld_vertices tolerance=0.01
}
```
-> 576 triangles, `isClosed` true, `boundaryEdges` 0, `signedVolume` 34,184,797,
no diagnostics. **Bore open.** Same volume, same closure, same orientation — 72
fewer triangles and a completely different object.

### Two intermediate probes, both from the same run

- **`capped=false` without the weld:** 576 triangles, `isClosed` **false**,
  `boundaryEdges` **72** — two open 36-edge rings. So the profile loop's repeated
  endpoint is **two distinct vertex rings, not one**: the closed-loop profile is
  never actually closed by the op, and the weld is what closes it. This is the
  measurement that identifies the mechanism.
- **Profile with the repeated endpoint dropped, `capped=false`:** 504 triangles,
  `boundaryEdges` 72, and `signedVolume` **122,690,095** — nonsense, since the
  surface is open across the section. Recorded so nobody reads that figure as a
  competing volume; an open mesh has no meaningful signed volume.

## The one-character fix, measured (2026-08-27, second agent)

**Do not repeat the endpoint, and leave `capped` alone.** A profile whose first
and last points do NOT coincide is closed implicitly by the op, emits no axis
cap, and needs no weld. Measured on `SM_Dome_Ruin_A.pwmodel`'s `band` part — a
7-point off-axis closed section, `steps=36`, `capped` **left at its default
`true`**:

| field | value | predicted |
|---|---|---|
| `meshTriangleCount` | 504 | **504** = 7 profile edges x 36 steps x 2 — no 72-triangle fan excess |
| `isClosed` | true | true |
| `boundaryEdges` | 0 | 0 |
| `signedVolume` | 612,065,343 | 639,400,000 analytic, less the erosion noise that runs after it |

The filled-bore volume for that same section would be **2.245e9**, so at
612,065,343 the bore is measurably open.

This contradicts nothing above: the 504-triangle row in the probes was taken at
`capped=false`, where the section really is left open across itself. At
`capped=true` the un-repeated endpoint is the sound spelling and **the repeated
one is the trap**. That strengthens fix (1) below — the op already welds a
non-coincident seam correctly, so the coincident case is the one handled wrongly,
not a missing capability.

## What it should do

In order of preference:

1. **Detect a profile whose first and last points coincide and weld that seam
   itself** rather than capping to the axis. The op demonstrably already does the
   right thing when the seam is implicit; only the explicit spelling of the same
   loop diverges.
2. Failing that, **warn when `capped=true` emits an axis cap at `angle=360` on a
   profile whose endpoint is not on the axis** — that is never what a full
   revolution wants, and the compiler has both facts in hand at the call site.
3. **Reword `capped`'s parameter text at `PwModelParser.cpp:561`** to say it caps
   to the **AXIS**, not to the ends of a partial sweep. Cheapest of the three and
   worth doing regardless of which of the other two lands.

## Workaround

`capped=false` plus `weld_vertices tolerance=0.01` (what `SM_Portal_Ring.pwmodel`
ships, written into its header so the next author does not simplify it out), or
— cheaper and now preferred — **do not repeat the profile's endpoint** and leave
`capped` at its default.

## Distinct from related tickets

- `B-extrude-polygon-uncapped-nonconvex` (IN-REVIEW, High) is the **exact
  inverse** and is worth reading beside this one: there `bCapped=true` produces
  **no** cap on a non-convex outline in the engine's
  `append_simple_extrude_polygon`; here `capped=true` produces a cap **nobody
  asked for** on a full `.pwmodel` `revolve`. Different verb, opposite direction
  of failure, opposite remedy. Not a duplicate — cited because a fixer touching
  cap generation should know both exist.
- `B-revolve-polygon-drops-material-id` (OPEN, Medium) is about the engine's
  `append_revolve_polygon` silently forcing `MaterialID` to 0. Same word
  "revolve", entirely different failure (slot assignment, not cap geometry), and
  it is the `geometry.*` RPC family rather than the `.pwmodel` op.
- A read-only sweep of the whole board this session found **no** other ticket on
  the `.pwmodel` `revolve` op.

severity rationale: impact=silent wrong geometry on a normal path — the membrane is invisible to every field the published health gate reads, `signedVolume` included, because the two axis caps cancel exactly, and `model.compile`'s own documented gate passes on it x reach=the standard way to author any ring / tube / rim / flange, and the parameter text actively misdirects the author away from the flag that controls it -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `revolve angle=360` (default) with a 9-point closed off-axis profile whose endpoint repeats the first, `capped` at default `true`: 648 triangles against the 576 predicted by 8 edges x 36 steps x 2, `isClosed` true, `boundaryEdges` 0, `orientationConsistent` true, `degenerateTriangles` 0, `signedVolume` 34,184,797 (correct), no diagnostics — and a flat membrane across the bore, visible only in the render (`Saved/Screenshots/AssetPreview/atl_portal_ring_q.png`, a solid disc with a six-wedge fan). The 72-triangle excess is exactly two 36-triangle axis fans; they are coincident and oppositely wound, so `signedVolume` cancels to the correct annulus figure and the published gate `health.isClosed && health.signedVolume > 0` passes. Two isolating probes: `capped=false` unwelded gave 576 tris, `isClosed` **false**, `boundaryEdges` **72** — two distinct 36-edge vertex rings, proving the repeated endpoint is never welded by the op; endpoint dropped at `capped=false` gave 504 tris, `boundaryEdges` 72 and a nonsense `signedVolume` 122,690,095 (open mesh, figure meaningless). Follow-up correction the same day from a second agent working `SM_Dome_Ruin_A.pwmodel`: **not repeating the endpoint is the one-character fix** — its 7-point `band` part at `steps=36` with `capped` left at default `true` gives exactly 504 = 7x36x2 triangles with no fan excess, `isClosed` true, `boundaryEdges` 0, `signedVolume` 612,065,343 against 2.245e9 for the filled-bore case, so the bore is measurably open. Documentation half source-confirmed at HEAD in this tree: `PwModelParser.cpp:561` publishes `capped` as "Cap the ends of a partial revolution", which reads as "only relevant when angle < 360" and is exactly why the flag was left at default; it in fact controls the caps to the AXIS. Geometry half NOT traced into compiler source — stated as inference from the measured responses. Worked around on `SM_Portal_Ring.pwmodel` with `capped=false` + `weld_vertices tolerance=0.01`; defect untouched.
- `#2-closed-section-revolves-through-polygon-generator` `IN-REVIEW` developer — Root-caused into the compiler, contradicting the ticket's "one-character fix": `FRevolvePlanarPathGenerator`'s ONLY route to a closed profile curve is its capping branch (RevolveGenerator.cpp:139-158), which appends the projections of the last and first points onto the axis and then sets `bProfileCurveIsClosed` — so a repeated endpoint gives two coincident, oppositely wound fans (the membrane), and `capped=false` gives two unwelded vertex rings. `GenerateRevolve` (Handlers/Geometry/GeometryOps_Primitives.cpp) now detects a profile whose first and last points coincide, drops the repeat IN `Params.Profile` (so `profilePoints` echoes the swept count), maps `capped` onto `RevolveOptions.bFillPartialRevolveEndcaps` — its documented partial-sweep meaning, ignored at 360 — and routes to `AppendRevolvePolygon` / `FRevolvePlanarPolygonGenerator`, which sets `bProfileCurveIsClosed` unconditionally and emits no axis points at all; it warns that it read the profile as a closed section. **Correction to the "do not repeat the endpoint" workaround recorded in #1: it is not a fix.** With the endpoint dropped the path generator still caps to the axis, and for a closed section the two cap points land at the same height, so the fans are still coincident, still cancel in `signedVolume`, and the 504-triangle figure quoted as "no fan excess" decomposes exactly as 6 quad edges x 72 plus two 36-triangle fans — 2*(N-1)*S for the repeated spelling equals (N-2) quad edges * 2S plus 2S of fan, so the triangle count cannot separate the two either. That spelling is ambiguous against a legitimate lathe and is therefore NAMED rather than repaired: a new warning fires when both profile ends sit off the axis at the same height under `capped`, giving the one-character remedy. `capped`'s parameter text reworded on both surfaces (PwModelParser.cpp `revolve` op, PrimitiveHandler.cpp `geometry.revolve`) to say it caps to the AXIS, not to the ends of a partial sweep. **The health gate is untouched and still cannot see a membrane** — `isClosed`, `boundaryEdges`, `orientationConsistent`, `degenerateTriangles` and `signedVolume` are green before and after, which is why the new tests assert on geometry. Tests: Tests/Model/TestPwModelRevolveClosedProfile.cpp — `PinWright.Model.Revolve.ClosedProfileLeavesTheBoreOpen` (no vertex reaches the axis; 96 triangles, not the 120 of the two fans), `.OpenProfileStillCapsToTheAxis` (a lathe is still a solid), `.ImplicitClosedSectionIsNamedNotRepaired` (the ambiguous spelling warns), `.ClosedProfileDocumentHasNoAxisFan` (the same profile through `model.validate`, asserting the count while recording that every health field passed on the broken mesh too).

- `#3-verified-fixed` `DONE` verifier — 2026-08-28. Plugin rebuilt from a clean tree at `b79ba53e` and verified against disk, not against the build's own success message: `UnrealEditor-PinWright.dll` 39,898,624 -> 40,644,096 bytes at 2026-08-28 08:11:48, `UnrealEditor-PinWrightGeometry.dll` 4,983,296 -> 5,113,344, canonical link with no `-000N` artifacts in `UnrealEditor.modules`. Editor restarted on that DLL and the ticket's own repro re-run. Bore confirmed open on the rebuilt DLL, against an OLD-DLL baseline captured first. The ticket's verbatim 9-point profile at `steps=36`: before, 648 triangles and zero diagnostics; after, **576 triangles** (8 swept edges x 36 x 2) with `signedVolume` 34,184,797.011 unchanged to the last digit, `isClosed` true, `boundaryEdges` 0, and a new `PWMODEL_STAGE_WARNING` stating the repeated endpoint is dropped and no axis cap is emitted. The 72-triangle fan excess is gone. Also checked the ambiguity #2 describes: an explicit `capped=true` on the same profile still yields 576, i.e. the closed-section route correctly ignores `capped` rather than honouring it and re-filling the bore.
