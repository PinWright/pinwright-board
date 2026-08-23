---
id: B-pwmodel-overlap-check-skips-modifiers
title: "PWMODEL_UNUNIONED_OVERLAP cannot fire for mirror or the three array_* ops — the diagnostic for the most-cited .pwmodel trap is unreachable on exactly the ops most likely to hit it"
status: OPEN
severity: Medium
category: bug
tags: [pwmodel, diagnostics, PWMODEL_UNUNIONED_OVERLAP, mirror, array_linear, array_radial, array_along_path, sweep, extrude_along_spline, unreachable-check, footprint]
encounters: 2
lastSeen: 2026-08-23T00:00:00Z
---

# The overlap warning is wired to the generator path only

`FCompiler::WarnOnUnunionedOverlap` has exactly **one** call site —
`Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:1062`, inside `FCompiler::RunGenerator`
(declared `:463`, defined `:919`, emit `:951`). A repo-wide grep finds no other caller, so the op
named in the message is always a generator.

`mirror`, `array_linear`, `array_radial` and `array_along_path` are all registered as **modifiers** —
`Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:962`, `:972`, `:977`, `:984`, built by
`MakeModifier` (`:155-158`), which does not set `Spec.bGenerator` (contrast `MakeGenerator`, `:148`).
`RunOp`'s routing gate at `PwModelCompiler.cpp:2046-2063` therefore sends all four to
`DispatchModifier` (`:1753`, `:1763`, `:1771`, `:1781`) and never to `RunGenerator`.

The comparison data is then destroyed: `PwModelCompiler.cpp:2071` runs `AppendedFootprints.Reset();`
inside the modifier branch (`:2066-2074`). One precision note on the original report — the reset runs
**after** `DispatchModifier` has already executed the op, not before it. The effect is the same (the
modifier's own multiplied geometry is never recorded as a footprint, and every earlier footprint is
dropped), but the ordering claim was not literally right. Sibling resets sit at `:2053` (boolean) and
`:2085-2086` / `:2098` (per-mesh scope save/restore); the design rationale is at `:2068-2070` and
`:483-486`.

## Why this is the wrong four ops to lose

All four **append** their copies rather than unioning them, so overlapping copies leave buried
interior faces and hand the next boolean a self-intersecting mesh — the exact condition the
diagnostic exists to name, and the single most-cited `.pwmodel` trap ("sibling ops APPEND, they do
not union").

## Coverage loss is wider than the four ops

Because the reset clears the whole list rather than skipping one entry, a **generator** that follows
any modifier also has nothing to compare against. `translate` then `cylinder` cannot raise the
warning either, even though `cylinder` runs the generator path.

## The documented fallback does not cover the case

`Docs/wiki-src/model.authoring.md:191` already states the limitation verbatim —
"`PWMODEL_UNUNIONED_OVERLAP` fires only from the generator path, and any modifier — which is what
`mirror` and the three `array_*` ops are — resets the footprint list the check compares against, so a
mirrored or arrayed copy can never raise it" — and the same fact is in
`Examples/pwmodel/pipe_junction.pwmodel:153-155`. Both offer `union { }` as the remedy and
`health.boundaryEdges` as the check. The second half of that advice is weak: two interpenetrating
**closed** solids appended together still report `boundaryEdges: 0`, so for the common case there is
no in-band signal at all. The wiki is honest that nothing reports it; the gap itself is open.

**Fix:** move the check out of `RunGenerator` so it runs for any op that appends geometry, and stop
clearing footprints wholesale on the modifier branch — a modifier that multiplies geometry should
*add* its copies' footprints, which is what would let `array_radial count=6` on an overlapping
profile say so. The closed-operand gate added in `5c32bd72` (which stopped the false positives on
open shells) is the right filter to keep on the widened path.

## Related

- `Docs/plans/defect-backlog.md` `D-67` (CONFIRMED) — `array_along_path` promises a bound the
  compiler does not apply. Same op, different defect.
- `Docs/plans/defect-backlog.md` `D-07` (CONFIRMED) — modifiers appending at material ID 0. Same
  append-semantics family, different consequence.

## History
- `#1-warn-wired-to-generator-path-only` `OPEN` reporter — `FCompiler::WarnOnUnunionedOverlap` has one call site, `PwModelCompiler.cpp:1062`, inside `RunGenerator` (decl `:463`, def `:919`, emit `:951`); no other caller exists in the tree. `mirror`, `array_linear`, `array_radial` and `array_along_path` are registered via `MakeModifier` (`PwModelParser.cpp:962/972/977/984`, helper at `:155-158`, which leaves `bGenerator` false), so the routing gate at `PwModelCompiler.cpp:2046-2063` sends all four through `DispatchModifier` (`:1753/:1763/:1771/:1781`), and the branch then clears the comparison data at `:2071` (`AppendedFootprints.Reset()`). Correction to the report: the reset runs *after* `DispatchModifier` executes the op, not before — same effect, different order. All four ops append rather than union, so overlapping copies bury interior faces and hand the next boolean a self-intersecting mesh, which is precisely what the diagnostic exists to name and the most-cited `.pwmodel` trap. The loss is wider than the four ops: because the reset clears the entire list, a generator following any modifier also has nothing to compare against. Documented as a known limitation at `Docs/wiki-src/model.authoring.md:191` and `Examples/pwmodel/pipe_junction.pwmodel:153-155`, both offering `union { }` plus `health.boundaryEdges` — but two interpenetrating *closed* solids appended together still report `boundaryEdges: 0`, so the documented detection signal provably does not detect the common case. Fix: run the check for any op that appends geometry rather than only from `RunGenerator`, and have multiplying modifiers add their copies' footprints instead of clearing the list; keep the closed-operand gate from `5c32bd72`, which is what removed the false positives on open shells. Not covered by any board ticket or by `D-07`/`D-67`.

- `#2-sweep-is-in-the-same-hole` `OPEN` reporter — independent confirmation, and the op list in the title is incomplete: `sweep` and `extrude_along_spline` are `MakeModifier` entries too (`PwModelParser.cpp`, the "Path-driven" block), so they take the same `DispatchModifier` branch and are equally unreachable. Measured on a document of **51 `sweep`s plus one `revolve`**, every swept tube `cap=true` and therefore closed, and most of them deliberately interpenetrating (limbs rooted INSIDE the trunk they grow from): `model.compile` reported `componentCount: 52`, `isClosed: true`, and **zero** `PWMODEL_UNUNIONED_OVERLAP` — total diagnostics 1, and that one was the unrelated `twist` orphaned-normal notice. The same shape built from `cone`s would have warned at nearly every joint. This matters beyond the missing warning: an author who has read the docs reasonably reads "no overlap warnings" as "no un-unioned overlaps", and on a sweep-built model that inference is unsound in the direction that ships buried interior faces. Note the check is structurally harder to widen for these two than for `mirror` / `array_*`: a sweep appends straight into the accumulated mesh with no scratch-mesh indirection, so there is no separable per-op box to record — widening it means routing appending modifiers through the same scratch mesh generators already use, or snapshotting the mesh and diffing, not just moving the call site.
