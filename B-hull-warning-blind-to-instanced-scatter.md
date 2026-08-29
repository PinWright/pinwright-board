---
id: B-hull-warning-blind-to-instanced-scatter
title: "`groundProvenance.warning` is gated on the ABSENCE of a face index, so any surface that answers with triangles — a complex-traced HISM scatter, a collisionless foliage mesh, a `UseComplexAsSimple` body — lands in `triangleColumns` and cannot raise it; the response's only affirmative statement about surface trustworthiness is structurally silent on the case it is most needed for, while `surfaceComponents[]` beside it already names the HISM"
status: IN-REVIEW
severity: High
category: bug
tags: [spatial, ground_actors, verify_grounding, ground_instances, provenance, face-index, hull, warning, hism, instanced-static-mesh, scatter, silent-wrong-data, level-building, placement, review-hazard]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The warning asks "did a column lack a face index", and the caller reads it as "did I measure the wrong surface"

`B-ground-probe-hits-hull-not-render` `#4` shipped `contact.groundProvenance`, and with it the one
field in the block that makes an affirmative claim rather than reporting a count: `warning`. It is
the field a caller stops at. Its predicate is `Prov.PrimitiveColumns > 0`
(`Source/PinWright/Private/Handlers/Spatial/GroundPlacementUtils.cpp:889`), and `PrimitiveColumns`
is incremented only in the `else` arm of a face-index test:

```cpp
// GroundPlacementUtils.cpp:518-525
if (Column.GroundFaceIndex >= 0)
{
    ++Report.Provenance.TriangleColumns;
}
else
{
    ++Report.Provenance.PrimitiveColumns;
}
```

The face index is copied off the hit at `:1022` (`Column.GroundFaceIndex = Hit.FaceIndex;`), beside
the two other provenance fields, inside the ground column probe.

So the warning does not fire on "you measured the wrong surface". It fires on "at least one column
came back without a face index". **Those are different sets, and an instanced scatter is routinely
in the first and not the second.**

## The two ways a scatter is actually probed both produce a face index

Neither of these is an exotic argument shape. They are the two calls the board's own DONE tickets
publish.

**1. `traceComplex: true`, the documented workaround.** `B-ground-probe-hits-hull-not-render`'s
own repro and its Workaround section both call
`spatial.verify_grounding {surface:{preset:"any_solid", traceComplex:true}, …}`, and the parser
honours it: `ApplyPreset()` runs first at `GroundPlacementUtils.cpp:331` (setting
`bTraceComplex = false` for `AnySolid` at `:243`), then an explicit caller field overrides it at
`:334-340`. Under a complex trace a HISM's render triangles answer, the hit carries a face index,
the column is counted as a `TriangleColumn`, and `PrimitiveColumns` stays `0`. **The warning cannot
fire on any column of that probe.**

**2. Collisionless foliage, which only a complex trace touches at all.** This is
`B-trace-complex-hits-render-geometry` (DONE) restated in provenance terms: **268 of 930** downward
probes on this project returned the height of a collisionless HISM tree. Every one of those is a
render-triangle hit, so every one is a `TriangleColumn`. `renderGeometryColumns` does rise
(`:526-528`, from `bRenderGeometryHit` = `bTraceComplex && SimpleCollisionShapes == 0`,
`SpatialTraceUtils.cpp:43`) — but nothing in the block turns that into a `warning`, and `warning` is
the field the parent ticket taught the caller to read.

There is a third route I could not settle in this tree and am recording as unproven rather than
asserting — see *"What I could not verify"* below: a body set `CTF_UseComplexAsSimple` answers even
a `traceComplex: false` query with per-triangle geometry, which this plugin states in its own words
at `Private/Utils/CollisionSummaryUtils.h:38` (*"CTF_UseComplexAsSimple - per-triangle geometry
answers simple queries"*) and encodes at `CollisionSummaryUtils.cpp:104-117`. If that holds through
a HISM's per-instance bodies, the blind spot reaches the **default** `traceComplex: false` path too.

## `any_solid` deliberately admits a plain scatter as ground, and hands the caller no signal to act on

This is not an accident of the preset — it is a stated refusal. `GroundPlacementUtils.cpp:233-238`:

> The generic instanced classes are deliberately NOT here. A HISM/ISM scatter of paving stones,
> rocks, debris or modular tiles is legitimate ground, and a preset that excluded
> `UInstancedStaticMeshComponent` wholesale would move every actor a caller had already seated on
> one. **That case is the caller's to state**, via the `excludeComponentClasses` list this preset
> shares.

The preset is right to refuse. But "the caller's to state" only works if the caller is told they
need to state it, and on both probe routes above the block's only alarm is provably silent while
every count in it reads clean.

## Two published sentences that are narrower in practice than they read

Quoted, not edited — both are cited from files another agent is working.

**`B-ground-probe-hits-hull-not-render` history `#4`:**

> The `warning` fires whenever any column was answered by a hull.

It fires whenever a column lacked a face index. Absence of a face index is not "answered by a
hull" — `SpatialTraceUtils.h:39-43` says so directly, listing `INDEX_NONE` for *"a simple-primitive
hit **or when the physics backend reported none**"*, and `#3` of that same ticket already recorded
the presumptive direction. This ticket is the **other** direction, which nothing has recorded yet:
*presence* of a face index is not "answered by the render surface" either, because a triangle
answer from a foliage scatter is exactly as wrong a ground as a hull answer from a chipped podium.

**The in-code comment guarding the gate, `GroundPlacementUtils.cpp:886-888`:**

> Name the trap in the response itself, the way `spatial.raycast` names its own. The wording holds
> under either `traceComplex` setting, because a primitive can answer a complex query too (a body
> set to use simple as complex, or a shape component).

The comment reasons about *primitive-answers-complex* and concludes the wording is safe under both
flags. It never considers *triangles-answer-and-the-surface-is-still-wrong*, which is the arm that
suppresses the warning rather than mis-wording it. The comment is correct about what it examined;
the gap is what it did not.

## What already knows the answer, and is not consulted

`surfaceComponents[]` — shipped by `E-ground-preset-excludes-only-foliage-actors` `#3` — carries a
`componentClass` per distinct answering primitive. Its own declaration names this exact case as the
worked example:

```cpp
// GroundPlacementUtils.h:367-370
FString ComponentClass; // its UClass name, e.g. "HierarchicalInstancedStaticMeshComponent"
```

and `:353-358` states the purpose: a caller should see a `HierarchicalInstancedStaticMeshComponent`
where they expected a `LandscapeHeightfieldCollisionComponent` *and know the SURFACE SPEC, not the
seat, is what needs fixing*. It is recorded per column, deduped by component pointer
(`GroundPlacementUtils.cpp:157-186`), aggregated at `:542`, sorted and capped at `:599-609`, and
serialized at `:867-883`.

**So the response contains the answer and the alarm is derived from a proxy that cannot see it.**
The warning is one `if` away from the component roster and does not read it.

## Fix direction, and what it costs

Derive the warning from `surfaceComponents[]` (component class) rather than from the presence of a
face index. The data is already assembled at the point the warning is written — `MakeProvenanceJson`
emits the roster at `:867-883` and the warning at `:889-893`, in that order, from the same `Prov`.

**Say the cost plainly: `surfaceComponents[]` is a per-row array, so a batch-level alarm needs an
aggregate over it**, and there is no such aggregate today — `SurfaceComponentCount` (`:883`) is a
distinct-primitive count, not a classification. A fixer has to decide what an aggregate over classes
means when a footprint straddles two surfaces, which the struct's own comment (`:373-376`) is
explicit is a distribution rather than a label and must not be collapsed. That is real design work,
not a one-line predicate swap, and pretending otherwise would set the next agent up to collapse the
distribution.

Two properties any fix should keep:

- **Keep the existing hull warning.** It is correct on the case it was built for and this ticket
  does not weaken it. The ask is a second condition, not a replacement predicate.
- **Do not turn "a scatter answered" into a failure.** Per the preset's stated refusal above, a HISM
  of paving stones is legitimate ground. The output is a `warning`, not a `failCode`.

## What I could not verify

Recorded so the next reader does not spend the same time. **Whether a `CTF_UseComplexAsSimple`
static mesh scattered as HISM/ISM instances returns a face index on a `traceComplex: false` probe is
NOT established here.** `UInstancedStaticMeshComponent::InitInstanceBody` builds per-instance bodies
from `GetBodySetup()` (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/InstancedStaticMesh.cpp:2840,
:2864, :2877`), and neither `InstancedStaticMesh.cpp` nor `HierarchicalInstancedStaticMesh.cpp`
mentions `CollisionTraceFlag` or `ComplexAsSimple` anywhere — a grep for either term in those files
returns nothing — so whether the physics backend cooks a triangle geometry for an instanced body of
a complex-as-simple mesh is not answerable by reading them. The ticket does not depend on it: routes
1 and 2 above are both source-verified and neither needs the flag. It matters only for **how wide**
the blind spot is, and it is the one experiment worth running before scoping the fix, because it
decides whether the default `traceComplex: false` path is affected.

Two things about that flag ARE established in this tree and are why it is worth checking rather than
dismissing: this plugin **ships a verb that sets it** —
`static_mesh.set_collision_complexity` (`Private/Handlers/Asset/StaticMeshSetCollisionComplexityHandler.cpp:68`
registers it, `:52` maps the token, `:116` writes `BodySetup->CollisionTraceFlag`) — so a caller can
put a mesh into that state through PinWright itself; and the plugin already models the inversion
correctly elsewhere (`CollisionSummaryUtils.h:102-105`, tested in
`Private/Tests/EditorOps/TestSetViewModeCollision.cpp:67-73`), so the two halves of the plugin
disagree about whether a simple query can be answered by triangles.

## Severity: High

**Impact class is the README's High band — "silent wrong … data on a normal path (the caller trusts
a result that is a lie and builds on it)".** The lie is the *absence* of the warning. `warning` is
the only field in `groundProvenance` that makes a claim rather than reporting a count; the parent
ticket shipped it precisely so a caller could stop reading, and on both probe routes above its
absence is guaranteed by construction regardless of what the probe actually hit. A caller who
follows `B-ground-probe-hits-hull-not-render`'s own published workaround — re-probe at
`traceComplex: true` — moves themselves onto route 1, i.e. **taking the published remedy is what
guarantees the alarm cannot fire.**

**The Medium counter-argument, weighed and declined.** Medium's band is "doable, but only via a
documented workaround" — and `surfaceComponents[]` does expose the truth, so a sufficiently careful
caller is not stuck. That is real and it is why this is not filed higher within its band. It is
declined because Medium describes an **omission** that forces a fallback, and this is an
**assertion**: the response does not fail to answer, it answers "no problem" over a real problem,
and the correcting datum is a per-row array with no threshold, no aggregate and no prose telling a
caller it supersedes the alarm beside it. Reading it is not a workaround, it is writing the check
the plugin declined to write — while the plugin's own published sentence (`#4` above) says the
alarm already covers it.

**`Critical` is not reachable**: it is defined by impact class alone — editor crash, or a write that
corrupts or loses asset data — and neither applies. Nothing crashes and no asset is corrupted; the
damage is a misplaced actor, recoverable once it can be seen. Same reasoning as
`B-ground-probe-hits-hull-not-render` `#2`.

**Reach modifier: neutral — I decline the bump DOWN for a rare edge path.** The instinct is that
this is exotic. It is not: `traceComplex: true` is the workaround a DONE High ticket publishes,
`any_solid` deliberately accepts plain instanced scatter as ground (`:233-238`), and
`B-trace-complex-hits-render-geometry` measured 268 of 930 probes landing on a collisionless HISM on
this one project. I also decline the bump **up**: the affected surface is the three grounding verbs
plus `foliage.paint` (all four route through `MakeProvenanceJson`), not every session, and terrain —
which is most probes — is unaffected, since a heightfield returns a face index and correctly raises
nothing. Common method, non-rare path, bounded blast radius: neutral both ways. **Severity `High`,
reach neutral.**

## Same shape as

`B-ground-probe-hits-hull-not-render`, `B-foliage-paint-does-no-ground-projection`,
`B-verify-grounding-maxgap-false-fail` — the call succeeds, every number it reports is correct, and
the output is wrong because the deciding number was never reported. The fullest statement of the
class is in `B-foliage-paint-does-no-ground-projection` `## Same shape as`; referenced, not
restated.

The distinguishing feature of *this* instance, and the reason it is not folded into the parent: the
deciding number **is** reported here (`surfaceComponents[].componentClass`). What is wrong is the
*summary* built on top of it, which is derived from a different, cheaper signal. That is a defect in
one predicate, not a missing measurement, and it can be fixed without touching the probe.

## Cross-links

- `B-ground-probe-hits-hull-not-render` (IN-REVIEW → being closed DONE this session) — the parent.
  `#4` built the `warning`; this is the coverage gap inside that fix, named as such. Nothing in that
  ticket is retracted: the hull-vs-render divergence, the 16 probe pairs, and the "always trace
  complex is NOT the fix" constraint all stand, and the warning it shipped is correct on the case it
  was built for.
- `E-ground-preset-excludes-only-foliage-actors` (IN-REVIEW) — shipped `surfaceComponents[]` in
  `#3`, i.e. the working detector this ticket asks the warning to be rebuilt on. Its `#2` also
  states the deliberate refusal to exclude generic ISM/HISM, which is the reason a response-side
  signal is required rather than a wider preset.
- `B-trace-complex-hits-render-geometry` (DONE) — the constraint that pins `traceComplex: false` as
  the preset default, and the source of route 2's 268/930 figure. Its fix (peeling via
  `onlyClasses`/`actorFilter`/`onlyActors`/`multiHit`) is about *which actor* answers; this is about
  the response saying *what* answered.
- `F-grounding-holder-not-seatable` (IN-REVIEW) — the same ISM/HISM family seen from the subject
  side rather than the surface side. Deliberately separate; a fixer could land either alone.

## History
- `#1-warning-predicate-is-a-proxy` `OPEN` reporter — Filed 2026-08-29 from live measurement on
  `EAContentExamples58` (UE 5.8), with every mechanism claim re-derived by `grep -n` / `sed -n`
  against HEAD in this tree rather than carried from the report. **Dedup sweep before filing.**
  `B-ground-probe-hits-hull-not-render` owns the divergence and shipped the `warning`, but its ask
  (`#2`, recast) was "serialize the provenance the probe already computes", which was done — this is
  a defect in the derived alarm, not a missing field, and closing the parent DONE does not close it.
  `E-ground-preset-excludes-only-foliage-actors` owns the FILTER and the component roster on the
  REPORT; it explicitly declines to exclude generic instanced classes, so it cannot own an alarm
  about them. `B-trace-complex-hits-render-geometry` (DONE) owns which actor answers.
  `F-grounding-holder-not-seatable` is the subject side. `B-foliage-paint-does-no-ground-projection`
  carries the class statement, referenced not restated. Also checked and unrelated:
  `B-static-mesh-missing-collision-body-counts`, `E-simplify-collision-decimates-render-mesh`,
  `B-geometry-convert-static-mesh-drops-collision`, `F-spatial-no-clear-footprint-search`. No
  existing ticket matches on `groundProvenance`, `primitiveColumns`, `triangleColumns` or
  `surfaceComponents`. Filed new. **Evidence:** two source-verified probe routes on which
  `PrimitiveColumns` is `0` by construction — an explicit `traceComplex: true` (parser ordering
  `GroundPlacementUtils.cpp:331` then `:334-340`, over the preset's `:243`) and the collisionless
  foliage path `B-trace-complex-hits-render-geometry` measured at 268/930 — against a warning gated
  solely on `:889`. A third route (`CTF_UseComplexAsSimple`) is recorded **unverified** in its own
  section with the engine files that failed to settle it, rather than asserted; the ticket does not
  rest on it. Severity `High` argued above against the rubric, with the Medium counter-argument and
  the declined reach-down both stated. **No umbrella:** this is one predicate in one function, and
  the recurring-class statement is referenced at `B-foliage-paint-does-no-ground-projection`.
  **Not RPC-verified this pass** — the routes are established from source and from two DONE
  tickets' own measurements; no `verify_grounding` call was made against a HISM scatter here, and a
  fixer should expect to construct that fixture (the existing
  `GroundProvenanceNamesTheAnsweringComponent` test in `Tests/Spatial/TestGroundPlacement.cpp`
  already builds a one-instance HISM surface and is the obvious place to hang it).
  **Concurrency note:** `B-ground-probe-hits-hull-not-render.md` is being edited by another agent
  and was read-only here — quoted, never modified.
- `#2-warning-rederived-from-the-component-roster` `IN-REVIEW` developer — "Re-gated the alarm on
  what the surface IS rather than on the absence of a face index, and published an affirmative
  verdict beside it. `MakeProvenanceJson` (`Handlers/Spatial/GroundPlacementUtils.cpp`) now builds
  a warning LIST: the existing hull sentence is kept verbatim as the first entry (the ticket's
  'second condition, not a replacement'), and three more can fire — an instanced scatter answered
  (`instancedScatterColumns`), render triangles answered because the struck component has no simple
  collision (`renderGeometryColumns`, which was counted and never warned about), and a
  `UseComplexAsSimple` body answered a `traceComplex:false` probe (`complexAsSimpleColumns`, gated
  on `!bTraceComplex` because under a complex probe per-triangle geometry is what was requested).
  All entries are joined into the SAME `warning` key rather than a new one, because `warning` is
  the field a caller stops at and a sibling key would have been as unread as the silence. Each
  sentence NAMES the offending surfaces from `surfaceComponents[]` (`actor.component [class]`), and
  says so when the capped roster cannot name them. **Nothing about what is traced changed** —
  `traceComplex:false` is still the default and the probe is untouched; per the fix direction this
  is a discard of already-computed provenance, not a trace-complexity change. **The distribution is
  not collapsed:** classification travels per row (`instancedScatter` / `renderGeometry` /
  `complexAsSimple`, emitted only when true, since the row's existence already says 'examined'), so
  a footprint straddling a scatter and a cliff keeps one answer per surface; the batch-level
  aggregate is a column COUNT per condition, never a label. **Rule-12 half:** new `surfaceTrust`
  (`trusted` / `untrusted` / `undetermined`) is published on every measured report. `undetermined`
  is reached when a supported column's primitive never resolved (new `unclassifiedColumns`, counted
  in `GroundNoteSurfaceComponent` where such columns were previously dropped silently) and NO
  condition fired — and in that state `warning` is OMITTED rather than written as an all-clear, so
  'measured and clean' and 'could not be measured' stop reading alike. Classification is done once
  per DISTINCT primitive (`Component->IsA<UInstancedStaticMeshComponent>()` covers HISM/ISM/foliage
  by ancestry; the trace flag goes through `PinWrightCollisionSummary::Summarize`, never the raw
  `CollisionTraceFlag`, so `CTF_UseDefault` resolves against the project default). **Regression
  test** `PinWright.spatial.verify_grounding.GroundProvenanceWarnsOnUntrustedSurface` in
  `Tests/Spatial/TestGroundPlacement.cpp`, in two halves. (1) Constructed columns, no physics:
  every column carries `GroundFaceIndex = 7`, so `primitiveColumns` is asserted 0 and the old
  predicate provably cannot fire under any argument shape — pointed first at a real HISM component
  and then at a real StaticMeshComponent flagged as a render-geometry hit, both of which must raise
  the warning and name the component; the two silences are pinned in the same block (clean surface
  -> `trusted` + no warning; unresolved primitive -> `undetermined` + no warning +
  `unclassifiedColumns`). (2) The live verb over the one-instance HISM at `traceComplex:true` — the
  parent ticket's own published workaround, i.e. the route that used to GUARANTEE silence — must
  report `untrusted` and name `HISM_GroundScatter`, and excluding the component class must take the
  scatter alarm away again so the alarm tracks what ANSWERED rather than what is in the level. Red
  today on both halves: `surfaceTrust` and `instancedScatterColumns` did not exist and the warning
  text never named a component. **Not verified by me:** NOT COMPILED and NOT RUN (orchestrator owns
  the build). The `CTF_UseComplexAsSimple` route is implemented but has NO test — every
  `/Engine/BasicShapes` mesh ships simple collision and mutating a shared engine `BodySetup` would
  corrupt it for the rest of the session (`Tests/Spatial/TestRaycastHandlers.cpp:247-253` records
  the same constraint), so a fixture needs a privately duplicated `UStaticMesh` that I judged
  out-of-scope risk for a no-compile wave; the ticket's own 'What I could not verify' section still
  stands unresolved. Wiki overlays updated for the new contract: `Docs/wiki-src/spatial.md`,
  `spatial.ground-placement.md` (both carried the now-narrower sentence 'a `warning` fires whenever
  any hull answered'), `foliage.md`. `check_test_ids.py`: CLEAN, 4759 ids, no dot-prefix collision.
  No error codes touched, no dump aspect affected."
