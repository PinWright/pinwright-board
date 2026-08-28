---
id: E-revolve-profile-winding-undocumented
title: "`revolve` never states which way a profile must be traversed, and the wrong way is a silently inside-out solid — no doc clause, and no diagnostic, where `extrude` already has both"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [pwmodel, revolve, winding, signed-volume, inside-out, missing-diagnostic, docs, lumen]
encounters: 1
lastSeen: 2026-08-27T18:55:42+05:00
---

# `revolve` profile traversal direction is undocumented, and the wrong direction produces an inside-out solid with no diagnostic

**Classification, stated plainly: this is a documentation + missing-diagnostic
defect, not a behaviour bug.** The winding rule is real geometry — the direction
a profile is walked in decides the facing of the surface it sweeps, and that is
correct and unavoidable. The defect is that **nothing in the plugin ever states
the rule, and nothing names it at the point of failure**, so every hollow
revolved form is a coin flip resolved only by spending a compile.

Traversed one way the solid comes out correct; traversed the other it is
uniformly inside-out. The compile is green either way: `success: true`,
`isClosed: true`, `boundaryEdges: 0`, `degenerateTriangles: 0`,
`nonManifoldVertices: 0`, `orientationConsistent: true`, `floatingCount: 0`, no
diagnostics. The one field that moves is `signedVolume`, which goes negative.

## Verbatim repro

One `.pwmodel`, two point orders, `model.validate` each. This is the hollow drum
of `SM_Dome_Ruin_A` — a closed off-axis section walked up the INNER face, over
the top, and down the OUTER face:

```
part shell {
    revolve steps=36 profile=[(1160, 380), (1160, 1200), (1120.5, 1500.2), (1004.6, 1780.0), (820.2, 2020.2), (580.0, 2204.6), (300.2, 2320.5), (101.1, 2355.6), (113.3, 2495.1), (336.5, 2455.7), (650.0, 2325.8), (919.2, 2119.2), (1125.8, 1850.0), (1255.7, 1536.5), (1300, 1200), (1300, 380)]
}
```

-> `signedVolume` **-1.73e9**. Reverse those same 16 points and it is
**+1.73e9**. Every other field in `health`, in `floatingGeometry` and in `bounds`
is identical between the two runs, and so is `meshTriangleCount`. Model-wide the
two spellings give **2,349,271,526** and **-2,674,361,388** (both parts negative
on the inner-first spelling).

## Root cause (guilty source lines — documentation and a missing check)

**1. The parameter text carries no winding clause.**
`Plugins/PinWright/Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:557-558`:

```cpp
        MakeRequired(TEXT("profile"), EPwModelParamType::PointList2,
            TEXT("Profile points [(x, y), …] revolved around the local Z axis.")),
```

That is the **complete** published documentation of `profile`, and it is what
`model.describe_ops {op:"revolve"}` serves. Nothing about direction.

**2. The prose page does not fill the gap either.** In this checkout's generated
`Saved/PinWright/wiki/model.authoring.md`, `revolve` appears on **11 lines**
(polygroups and `bevel` safety, the `steps` floor at `angle=360`,
`harmonic_deform` aliasing, `sweep`'s `profile=` sharing the same list type,
`center=` sitting on the revolve axis). None of them mentions profile direction.
*(The session log recorded "five times"; the count measured against this tree
today is 11. The substantive claim — that none of them covers winding — holds
either way.)* The same page **does** carry a full winding treatment for
`extrude` (`:233`) and for `mirror` (`:245`), and `model.compile.md:57` explains
`signedVolume` as the inside-out signal and names
`PWMODEL_EXTRUDE_FACING_OPPOSED` as the authoring-time catch — so the house
convention for exactly this problem exists and `revolve` is simply outside it.
`Saved/PinWright/wiki/geometry.revolve.md` has no winding clause either.

**3. Both worked examples in the corpus teach the wrong-shaped rule.** The
`driftwood` example and the sibling `SM_Column_Broken_A.pwmodel` in this build
both use profiles that **START ON THE AXIS** and run outward. That is a
different-looking figure from the closed off-axis loop an author writes for a
hollow shell, so **copying the example does not transfer the rule** — which is
what makes this a trap rather than a lookup.

## `extrude` already has exactly this treatment, one op over

This is the strongest argument for the fix: the same class of mistake on the
neighbouring op is already caught at its line, with a message that names the
cause and the remedy. Verified in this tree at HEAD.

Declared, with its rationale, at
`Source/PinWrightGeometry/Private/Model/PwModelDiagnostic.h:264` (comment
`:253-263`):

```cpp
    inline constexpr TCHAR PWMODEL_EXTRUDE_FACING_OPPOSED[] = TEXT("PWMODEL_EXTRUDE_FACING_OPPOSED");
```

Raised at `Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:1772-1783`:

```cpp
    Warn(PwModelDiagnosticCodes::PWMODEL_EXTRUDE_FACING_OPPOSED, Op,
        FString::Printf(
            TEXT("'extrude direction=(%g, %g, %g)' points AGAINST this surface's facing normal ")
            TEXT("(%.3f, %.3f, %.3f) - dot %.3f - so the resulting shell will be inside out. …")
            TEXT("Check health.signedVolume: it is negative on an inverted closed shell."),
```

The header comment on `:262-263` states the reason it is raised at the call site
rather than downstream — *"nothing downstream can: the resulting shell is closed,
manifold, 0 boundary edges, and renders identically to a correct one. This is the
authoring mistake that shipped an inside-out example part for two releases."*
**Every word of that applies verbatim to `revolve`.**

## Lighting consequence — why no capture finds it

Backface culling shows whichever wall faces the camera, so an inverted revolved
shell **renders identically from every angle**. No asset-preview capture at any
orientation finds it. But offline consumers read winding, not shading: the mesh
distance field decides inside/outside by counting backface hits, so an inverted
shell inverts its field and **Lumen and DFAO light the part as though the camera
were inside it** (the mechanism is already written up at
`model.authoring.md:241`). On this project — `bGenerateMeshDistanceFields` on,
Lumen on — that is a real lighting defect that is visible only in the lit level
and invisible in every capture-based verification.

## What it should do

1. **One clause on the `profile` parameter text** at `PwModelParser.cpp:558` —
   that is where an author looks and, being served by `model.describe_ops`, it
   cannot drift from what compiles. Suggested rule as measured: walk the profile
   so the enclosed section stays **on your left** in the (r, z) plane with r
   right and z up — equivalently, up the OUTER face, over the top, back down the
   INNER face.
2. **A line-anchored warning on the `revolve` op itself when the part it produced
   came out with negative `signedVolume`**, naming profile direction as the
   cause. The compiler already computes per-part `signedVolume`, so the fact is
   in hand; this is the `revolve` twin of `PWMODEL_EXTRUDE_FACING_OPPOSED` and
   should read like it.

## Workaround

Traverse the profile up the OUTER face, over the top, and back down the INNER
face. Read the **per-part** `signedVolume`, not just the model-wide one —
`model.compile`'s page already warns that a model-wide sum averages one inverted
part away, and a building with a separate stained-stone base band is exactly that
shape.

## Distinct from related tickets

- `F-sweep-per-frame-scale-law` (IN-REVIEW, Medium) is the **closest class match
  on this board** and is cited as precedent, not as a duplicate. It names the
  identical failure shape — an operation that silently produces reversed facing
  normals while `isClosed`, `boundaryEdges` and the triangle count all stay
  identical, catchable only after the fact by `health.signedVolume` — and it
  closed that hole by **holding rather than extrapolating** `scales=` outside its
  outermost knots. Different op (`sweep` / `extrude_along_spline`), already
  fixed, and it carries **no revolve clause**. What it establishes is that this
  project already treats "silently reversed facing with a green health block" as
  worth engineering against.
- `B-revolve-closed-profile-fills-bore` (OPEN, High) is the sibling `revolve`
  defect found in the same session: `capped=true` at `angle=360` on a closed
  off-axis profile emits axis caps that fill the bore. Different failure (extra
  geometry vs. reversed facing), different remedy, and its `signedVolume` reads
  **correct** where this one reads **negative** — the two are distinguishable by
  exactly that field. Fix them together while the op is open.
- `B-revolve-polygon-drops-material-id` (OPEN, Medium) is the engine's
  `append_revolve_polygon` forcing `MaterialID` to 0 in the `geometry.*` RPC
  family. Word collision only.

severity rationale: impact=soft blocker — the failure is reachable and recoverable, and `model.compile`'s published gate `health.isClosed && health.signedVolume > 0` does catch it, but only by spending a compile cycle, with no way to learn the rule beforehand and nothing naming the remedy at the point of failure x reach=every hollow revolved form (dome, drum, vase, bell, rim, shaped-wall pipe), and the failure renders identically at every angle so no capture-based verification finds it — on a Lumen + mesh-distance-field project it ships as a lighting defect -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Found building `/Game/Atlantis/Meshes/SM_Dome_Ruin_A` on the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. Same 16-point closed off-axis profile, two traversal orders, `steps=36`: `signedVolume` **-1.73e9** vs **+1.73e9** with every other field in `health`, `floatingGeometry` and `bounds` identical and the same `meshTriangleCount`; model-wide 2,349,271,526 against -2,674,361,388. Classified honestly as a documentation + missing-diagnostic defect, not a behaviour bug — the winding rule is real geometry. Source-confirmed in this tree: `PwModelParser.cpp:557-558` publishes `profile` in full as "Profile points [(x, y), …] revolved around the local Z axis" with no winding clause, and that string is what `model.describe_ops {op:"revolve"}` serves; `Saved/PinWright/wiki/model.authoring.md` mentions `revolve` on 11 lines (log said five — recounted against this tree) and none covers profile direction, while the same page carries full winding treatments for `extrude` (`:233`) and `mirror` (`:245`); `geometry.revolve.md` has no winding clause either. Both corpus examples (`driftwood`, `SM_Column_Broken_A.pwmodel`) use profiles that START ON THE AXIS, a different-looking figure from the closed off-axis loop a hollow shell needs, so copying them does not transfer the rule. Precedent verified at HEAD: `extrude` already has this exact treatment via `PWMODEL_EXTRUDE_FACING_OPPOSED` — declared `PwModelDiagnostic.h:264` (rationale comment `:253-263`, "nothing downstream can … shipped an inside-out example part for two releases"), raised `PwModelCompiler.cpp:1772-1783` with a message that names the cause, the remedy and `health.signedVolume`. Lighting consequence recorded: backface culling makes the inverted shell render identically from every angle so no capture finds it, while the mesh distance field inverts and Lumen/DFAO light the part as if the camera were inside (mechanism at `model.authoring.md:241`); on this project that is a real defect visible only in the lit level. Cost one compile cycle; worked around by reversing the traversal. Defect untouched.
- `#2-winding-clause-and-diagnostic` `IN-REVIEW` developer — "Added the winding clause to `revolve`'s `profile` text in PwModelParser.cpp (x is the radius, y the height, traverse COUNTER-CLOCKWISE - up the OUTER face, over the top, back down the INNER face) and the `extrude` twin the ticket asked for: PWMODEL_REVOLVE_PROFILE_REVERSED, declared in PwModelDiagnostic.h and raised from FCompiler::WarnOnRevolveProfileReversed in PwModelCompiler.cpp at the revolve's own line whenever the mesh the op produced is closed and encloses negative signedVolume, naming the direction, the remedy and the Lumen/DFAO consequence. Measured on the produced mesh rather than on the point list so the closed off-axis section, the axis-capped lathe and a `capped` partial sweep are one predicate; quiet on an open result (signed volume means nothing there) and on a negative-determinant `scale=`, which reverses winding on its own and would make the warning blame the wrong parameter. Prose added under `## Primitive and response readings that are not what the name suggests` in Docs/wiki-src/model.authoring.md next to the existing `extrude` winding cluster, and a catalog row in Docs/pwmodel-format.md (required: PinWright.core.pwmodel_diagnostics.DocumentedCodesMatchEmittedCodes fails on any registered code with no table row). Regression test PinWright.Model.Revolve.ReversedProfileIsNamedAtItsLine in TestPwModelRevolveClosedProfile.cpp compiles the same four-corner ring section both ways round, asserts every published count is identical and only signedVolume flips sign, asserts the code fires on line 3 of the reversed document with COUNTER-CLOCKWISE in its message, and asserts the correct winding stays silent. NOT COMPILED OR RUN - build/test verification is the orchestrator's."
