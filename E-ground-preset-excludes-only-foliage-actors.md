---
id: E-ground-preset-excludes-only-foliage-actors
title: "The any_solid preset's foliage guard is one actor-class exclusion (InstancedFoliageActor), and FSpatialHitFilter::Matches only ever sees an AActor — so foliage carried as owned HISM components on an ordinary actor is invisible to both halves of the guard, and 268 of 930 probes on this project seated against it"
status: IN-REVIEW
severity: High
category: ergonomic
tags: [spatial, ground_actors, verify_grounding, any_solid, surface-preset, foliage, hism, instanced-static-mesh, hit-filter, silent-wrong-data, level-building, placement]
---

# The guard protects only the road you cannot take

`ESurfacePreset::AnySolid` advertises itself, in the `surface` parameter schema, as *"anything
blocking except foliage actors and effect geometry"* (`Handlers/Spatial/GroundPlacementHandler.cpp:717-719`).
Its foliage half is exactly one line:

```cpp
Filter.ExcludeClasses.AddUnique(TEXT("InstancedFoliageActor"));
```

`Handlers/Spatial/GroundPlacementUtils.cpp:151`, in the `AnySolid` case at `:137-154`. The comment
above it (`:138-143`) states the intent plainly: *"Foliage is excluded by class because its
instanced components are collisionless yet still block a complex query."*

## Both halves of the guard are structurally blind to owned HISM

**The class exclusion is actor-only.** `FSpatialHitFilter::Matches` takes an `AActor*` and nothing
else (`Handlers/Spatial/SpatialTraceUtils.cpp:152`), and `ExcludeClasses` is applied by
`SpatialTraceMatchesAnyClass(Actor, ExcludeClasses)` at `:198` — actor class ancestry, per `:51`.
The filter has **no component-class axis at all**. Foliage placed as
`UHierarchicalInstancedStaticMeshComponent`s on an ordinary `AActor` therefore never matches
`InstancedFoliageActor`, because the actor is not one; only the components are foliage.

**The intrinsic exclusion misses it too, for a different reason.**
`bExcludeEffectGeometry` (`SpatialTraceUtils.h:117`, applied at `SpatialTraceUtils.cpp:224`) is the
preset's other guard, and it *does* walk components — `IsEffectGeometryActor` at `:232-264`
enumerates every `UPrimitiveComponent`. But it deliberately skips components whose collision is
disabled (`:244-250`: *"A component a trace cannot hit cannot be the thing that was mistaken for
ground, so it gets no vote either way"*) and returns `CollidableCount > 0` (`:263`). An actor whose
only primitives are collisionless HISM foliage therefore returns **false** — not effect geometry —
and passes the filter as ground. The two exclusions fail on the same input for opposite reasons: one
looks only at the actor, the other looks at components but abstains on exactly these ones.

So a hit on owned HISM foliage is reported as a ground contact, with a Z the caller then seats a
prop on. Every number in the response is correct about the trace that was run; nothing in it says
the surface was foliage the preset claimed to exclude.

## Why the missed route is the *only* route

The `foliage.*` namespace writes into the level's shared `AInstancedFoliageActor`
(every write verb routes through `GetOrCreateFoliageActorForWorldSafe`, `Handlers/Environment/FoliageHandler.cpp:40`, called at `:214` by `foliage.paint` and `:875` by `foliage.add_instances`), so a foliage layer is not
prefix-addressable, not deletable, and not rebuildable as a unit. The plugin says as much itself, in
the hint `spatial` emits when a scatter holder is handed to a ground verb: *"a non-foliage ISM/HISM
has no per-instance write verb here and has to be re-scattered by whatever built it"*
(`GroundPlacementUtils.cpp:516-527`). Builders who need an addressable, rebuildable vegetation layer
therefore reach for owned HISM — which is the shape the guard cannot see.

**Measured on this project:** 268 of 930 downward probes resolved against a foliage mesh carried by
a HISM component, none of which the class exclusion could match. That is 29% of a real grounding
pass, on a preset whose stated purpose is to prevent exactly that.

**One honest qualification.** The preset also sets `bTraceComplex = false`
(`GroundPlacementUtils.cpp:153`), so a genuinely collisionless HISM does not block a *simple* probe.
The class exclusion is therefore belt-and-braces for two supported cases: a caller who sets
`traceComplex: true` (a documented knob on the same `surface` object, `GroundPlacementHandler.cpp:721-724`),
and a foliage mesh that carries real simple collision — common on kelp, coral and prop-grade
vegetation, and the case this project hit. The point of this ticket is not that every collisionless
HISM blocks; it is that in **every** case where a foliage surface does answer a probe, the guard
that exists to catch it cannot.

## Fix: a sibling intrinsic predicate, not a longer class list

Adding `HierarchicalInstancedStaticMeshComponent` to `ExcludeClasses` does not work — the filter
compares *actor* classes, so the entry would never match, and widening it to plain `AActor` would
exclude everything.

The right shape already exists next door. `bExcludeEffectGeometry` is an **intrinsic** test: it
answers a question about the actor from its components rather than from a name or a class the caller
must know (the comment at `GroundPlacementUtils.cpp:144-149` records why the name-based version was
removed — *"the wildcard name `FG_*`, which is one project's prefix"* — and that reasoning applies
verbatim here). Add its sibling:

- `bExcludeInstancedVegetation` (or a component-class axis on `FSpatialHitFilter`, if a general
  `ExcludeComponentClasses` is judged more useful — decide once, since the filter is shared by
  `spatial.ground_actors`, `spatial.verify_grounding` and `spatial.raycast`), set by `AnySolid`
  alongside `bExcludeEffectGeometry`.
- Answered from the hit's own component, not from the actor. **This needs a signature change**:
  `Matches(AActor*)` cannot express it. The hit already carries the component; pass it through.
- Exposed on the `surface` object as `excludeInstancedVegetation?` so a caller whose ground genuinely
  *is* an instanced mesh can turn it off — the same escape hatch `excludeEffectGeometry` has.

And, independent of which shape wins: **report the deciding fact.** The ground verbs should name the
hit component and its class in the per-actor result, so a caller can see that their "ground" was a
HISM and not stone. Without that, the next blind spot in this filter is as invisible as this one.

## The documented workaround, and why it does not demote this

`excludeClasses` is caller-settable on the `surface` object and *adds* to the preset
(`GroundPlacementHandler.cpp:713-714` schema, `:254-255` parse; `GroundPlacementUtils.cpp:286-287`),
so a project whose scatter holders share a distinct actor class — a `BP_KelpScatter`, say — can
exclude them cleanly today. Where the holders are plain `AActor` spawns, as they are here, no class
name distinguishes them from every other actor and the only remaining lever is `excludeNames` with a
prefix — which is precisely the one-project-prefix anti-pattern this file already removed once, for
reasons it records in the comment cited above.

The workaround also requires the caller to already know that `any_solid`'s foliage guard is
incomplete. That knowledge is what this ticket exists to supply, so it cannot also be the reason the
ticket is minor.

## Cross-links

- **`B-ground-probe-hits-hull-not-render`** (IN-REVIEW, High) — **the same 25 lines**. It cites
  `GroundPlacementUtils.cpp:134` and `:153` (the two presets' `bTraceComplex = false`) for a
  different axis: simple-vs-complex collision, hull vs render mesh. It does not mention the class
  exclusion at `:151`. Anyone fixing either will be editing inside the other's citations; read both
  before touching `ApplyPreset`.
- **`F-ism-per-instance-transforms`** (IN-REVIEW, High) — the addressability gap that makes owned
  HISM the route builders take. **Note for a fixer:** its three verbs (`actor.get_instances`,
  `actor.set_instance_transforms`, `spatial.ground_instances`) are **not present in this checkout** —
  `grep` for them across `Source/PinWright/Private/Handlers/` returns nothing. That work is
  IN-REVIEW on a sibling host. If `spatial.ground_instances` shares `FGroundSurfaceSpec`, as its
  ticket describes, it inherits this defect and hits it harder: grounding instances against a
  neighbouring scatter is the exact case.
- **`B-ism-undo-record-unsafe`** (OPEN, High) — same verb family, different axis.
- **`B-ortho-capture-culls-distant-foliage`** (OPEN, High) — the other place foliage is invisible to
  a subsystem that believes it is looking at it.

## Not RPC-verified

The 268/930 figure is from a measured grounding pass on this project, recorded before this review.
The mechanism above is source-read: the editor was not running for this pass, and no probe was
re-run to confirm which of the two blocking causes (opted-in `traceComplex`, or real simple collision
on the foliage mesh) applied to those 268. An editor test would settle that, and would also settle
whether a component-class axis is enough or whether the density of foliage in a scatter needs a
different answer. Neither changes the structural finding: `Matches` sees only an `AActor`.

## A note on this ticket's prefix

Filed as `E-` per the id it was proposed under. The defect reads as a bug — an advertised guard that
does not hold on a normal path — and a reviewer may reasonably prefer it re-filed as `B-`. Recorded
here rather than acted on, since the board's category is derived from the prefix and a duplicate id
would be worse than a mislabelled one.

severity rationale: impact=High — silent wrong data on a normal path: a foliage surface the preset promised to exclude is reported as a ground contact, the caller seats a prop on it, and no field in the response distinguishes that hit from stone × reach=normal — grounding is a core level-building loop and `any_solid` is one of only three presets, so no modifier applies; the Medium case is real and is being declined deliberately: `excludeClasses` is a documented, caller-settable workaround, but it is clean only where scatter holders share a distinct actor class, degrades to the prefix-matching anti-pattern this file already removed when they do not, and in every case requires the caller to already know the guard is incomplete -> High

## History
- `#1-any-solid-misses-owned-hism` `OPEN` reporter — Source-read only, editor not running; the 268/930 figure is from an earlier measured grounding pass on this project and was not re-run. `ESurfacePreset::AnySolid` implements its advertised foliage exclusion as a single actor-class entry, `Filter.ExcludeClasses.AddUnique(TEXT("InstancedFoliageActor"))` (`GroundPlacementUtils.cpp:151`), but `FSpatialHitFilter::Matches` takes an `AActor*` (`SpatialTraceUtils.cpp:152`) and applies `ExcludeClasses` by actor-class ancestry only (`:198`, `:51`) — there is no component-class axis, so foliage carried as owned HISM on an ordinary actor never matches. The preset's other guard misses it independently: `IsEffectGeometryActor` (`SpatialTraceUtils.cpp:232-264`) does walk components but abstains on collision-disabled ones (`:244-250`) and returns `CollidableCount > 0` (`:263`), so a collisionless-HISM-only actor is classified "not effect geometry" and passes as ground. Owned HISM is the route the `foliage.*` addressability gap leaves — the plugin says so itself at `GroundPlacementUtils.cpp:516-527` — so the guard covers only the road not taken. Qualification recorded in the body rather than glossed: the preset pins `bTraceComplex = false` (`:153`), so the class exclusion is belt-and-braces for a caller who sets `traceComplex: true` or for foliage meshes carrying real simple collision, which is this project's case. Dedup: searched the board for `any_solid`, `ExcludeClasses`, `InstancedFoliageActor`, `HISM`, `hit filter`, and every `B-ground-*` / `E-ground-*` / `spatial` ticket. `B-ground-probe-hits-hull-not-render` (IN-REVIEW) cites the same `ApplyPreset` block — `:134` and `:153` — but for the simple-vs-complex-collision axis and never mentions the class exclusion at `:151`; not a duplicate, and cross-linked as mandatory reading for a fixer since both edits land in the same 25 lines. `B-ground-actors-prefix-captures-foreign-actors` is about actor *selection*, not surface filtering. `B-verify-grounding-maxgap-false-fail` is a threshold defect. `B-ortho-capture-culls-distant-foliage` and `F-ism-per-instance-transforms` are cross-linked, not duplicated. Confirmed against this tree rather than trusting the board: `F-ism-per-instance-transforms` is IN-REVIEW, but `actor.get_instances`, `actor.set_instance_transforms` and `spatial.ground_instances` return zero grep hits under `Source/PinWright/Private/Handlers/` here — that work lives on a sibling host, and if `spatial.ground_instances` shares `FGroundSurfaceSpec` it inherits this defect.
- `#2-component-class-axis` `IN-REVIEW` developer — **Not stale: every structural claim re-verified at HEAD**, by symbol search rather than by the ticket's line numbers, which had all moved. `ApplyPreset`'s `AnySolid` case still carried exactly one foliage guard, `Filter.ExcludeClasses.AddUnique(TEXT("InstancedFoliageActor"))` (now `GroundPlacementUtils.cpp:173`); `FSpatialHitFilter::Matches` still took `AActor*` and nothing else; `ExcludeClasses` was still applied by `SpatialTraceMatchesAnyClass` walking ACTOR class ancestry; `IsEffectGeometryActor` still abstained on collision-disabled components and returned `CollidableCount > 0`. Both halves of the guard were blind to an owned instanced scatter, as filed. **Yesterday's work does not answer it and does not supply the predicate.** `FindInstancedHolder`/`DescribeInstancedHolder`, `HOLDER_NOT_SEATABLE`, `MeasureContactForBounds` and `spatial.ground_instances` are all present now, but they are about the SUBJECT being seated, never the SURFACE filter — `FindInstancedHolder(const AActor*)` asks "are this actor's bounds dominated by a scatter of >= 2 instances", which is actor-scoped and count-gated, and reusing it as the surface predicate would be wrong both ways (an actor owning real ground plus a decorative scatter would be excluded entirely; a single-instance ISM would not be). **Fix: a component-class axis, the input the filter never had.** Added `FSpatialHitFilter::ExcludeComponentClasses` (`SpatialTraceUtils.h`), folded it into `IsEmpty()`, and changed the signature to `Matches(AActor*, const UPrimitiveComponent* Component = nullptr)`; extracted the class-ancestry walk into a class-kind-agnostic `SpatialTraceClassChainMatches(const UClass*, ...)` so the actor and component axes cannot drift on what "matches a class" means, and passed the struck primitive at both production call sites (`TraceLineLayered` uses `Hit.HitComponent`, `ProbeFootprintOccupancy` uses the overlap's component). The default argument keeps every existing caller compiling. A hit with no resolvable component is never rejected on this axis — an unknown is not evidence. Wired `excludeComponentClasses` (+ `exclude_component_classes`) through BOTH surface parsers, which are still duplicated (`GroundPlacement::ParseSurfaceJson` and the handler-local `GroundRpcParseSurface`), plus `GroundRpcSurfaceEcho` and the two enumerating schema strings; `spatial.verify_grounding` defers to `ground_actors`' text and `spatial.raycast` exposes no exclusion axis at all, so neither needed a change. **Widening `any_solid` was scoped deliberately, and the declined half is the important part.** The preset now also excludes `FoliageInstancedStaticMeshComponent` and `GrassInstancedStaticMeshComponent` — two engine classes that exist for vegetation and nothing else, both deriving straight from `UHierarchicalInstancedStaticMeshComponent` so neither covers the other by ancestry and both must be listed. It does **not** exclude plain `InstancedStaticMeshComponent`/`HierarchicalInstancedStaticMeshComponent`, and that is a refusal, not an oversight: nothing intrinsic separates a HISM of grass from a HISM of paving stones, so a preset that excluded every instanced component would silently relocate every actor any caller had already seated on a scatter — the ticket's own 268/930 is the size of that behaviour change. So the reported case is only fixed by default where the scatter uses a foliage/grass component class; a plain-HISM scatter is now *expressible* (`excludeComponentClasses:["HierarchicalInstancedStaticMeshComponent"]`) where before it was expressible on no axis at all — `excludeClasses` cannot name an ordinary `AActor` holder and `excludeNames` degrades to the one-project-prefix anti-pattern this file already removed once. As shipped the change is non-breaking: the new axis defaults empty, and the preset's two additions only make the guard fire where it already claimed to. **Regression test** `Private/Tests/Spatial/TestInstancedSurfaceFilter.cpp`, three tests, both directions: `PinWright.spatial.surface.FoliageComponentOnOrdinaryActorIsExcluded` (a `UFoliageInstancedStaticMeshComponent` scatter on a plain `AActor` — asserts the holder is NOT an `AInstancedFoliageActor`, that `any_solid` rejects it, and that a filter carrying only the actor-class entry ACCEPTS the identical hit, so "the class list happened to match" cannot pass for the fix); `PlainInstancedScatterIsStillGround` (the non-breaking guarantee — `any_solid` still accepts a plain HISM, the preset carries no generic instanced entry, a caller-supplied entry excludes the same scatter, and the exclusion stays scoped to the component rather than condemning the actor; this is the assertion the natural over-fix fails); `SurfaceJsonCarriesComponentClassAxis` (both wire spellings reach the filter, caller entries ADD to the preset, `custom` implies nothing, and a filter carrying only the new axis is not `IsEmpty()` — which would otherwise short-circuit to accept-everything). Fixture classes are resolved by reflection (`UFoliageInstancedStaticMeshComponent` is `MinimalAPI`), and both editor-world tests emit `PINWRIGHT_ASSERTIONS_SKIPPED` through the shared emitter. **Not compiled and not run** — per the wave's instruction, the build and suite happen after. **Deliberately not done**, and still open as an ergonomic follow-up worth its own ticket: the body's second ask, that the ground verbs NAME the hit component and its class in the per-actor result. It touches the per-column report struct and the response emit, which is a different change from the filter, and without it the next blind spot in this filter is as invisible as this one was. Docs: `Docs/wiki-src/spatial.ground-placement.md` gained the axis in the preset list and a "What 'foliage' means" section stating the declined generic-ISM case explicitly (kept above the file's first `###`, of which it has none). Overlap check: `B-foliage-paint-does-no-ground-projection` is in `Handlers/Environment/FoliageHandler.cpp` and did not touch anything here; `B-ground-probe-hits-hull-not-render` cites the same `ApplyPreset` block for the `bTraceComplex` line, which this change leaves byte-identical.
- `#3-report-names-the-answering-component` `IN-REVIEW` developer — The body's second ask, deferred by `#2-component-class-axis` because it touches the report struct rather than the filter, now landed. **Re-verified at HEAD by symbol search, not by the ticket's line numbers, which had moved again.** `#2`'s filter work is intact — `ExcludeComponentClasses`, `Matches(AActor*, const UPrimitiveComponent*)`, the two vegetation entries on `AnySolid`, both wire spellings in both surface parsers — and the report side was still exactly as `#2` left it: `FGroundColumn` carried `GroundActor` but not the struck component, so `FGroundProvenance` could count which collision REPRESENTATION answered and nothing could say the representation belonged to a scatter rather than to a cliff. The ground actor is not published either, so a caller had no field at all naming the surface. **The change is one axis, threaded through the three structs the ticket names, and emitted from ONE site.** `FGroundColumn::GroundComponent` (a `TWeakObjectPtr<UPrimitiveComponent>`) is assigned from `Hit.HitComponent` in the probe loop beside the three provenance fields already read off that hit; `AggregateColumns` folds it into a new `FGroundProvenance::SurfaceComponents` — one row per DISTINCT primitive (`{Component, ActorLabel, ComponentName, ComponentClass, Columns}`), deduped by **component pointer identity** rather than by name, for the same reason `MeasureContact` keys its rejected-actor seen-set on the actor and not on its label; `StableSort` descending by tally so two runs over one world produce the same list; capped at 8 with the true distinct total in `SurfaceComponentCount`, mirroring `rejectedSurfaceActorCount` / `rejectedSurfaceActors`. **Deliberately a distribution, not a label:** a footprint may straddle two surfaces and collapsing that would invent an answer for whichever half lost. A column whose component does not resolve contributes no row — an unknown is not a surface, and `supportedColumns` minus the summed tallies recovers that count honestly. **All three verbs get it from one site**, as required: `spatial.verify_grounding`, `spatial.ground_actors` and `spatial.ground_instances` all serialize through `GroundRpcContactObject`, which emits `groundProvenance` once, gated on `SupportedColumns > 0`. The wire shape is `surfaceComponents: [{actor, component, componentClass, columns}]` + `surfaceComponentCount`, sitting inside `groundProvenance` beside the counts it explains rather than as a parallel block. **The provenance writer itself moved** out of `GroundPlacementHandler.cpp`'s anonymous namespace to `GroundPlacement::MakeProvenanceJson` (`GroundPlacementUtils.h/.cpp`) so `foliage.paint` could publish the identical block — see `B-foliage-paint-does-no-ground-projection` `#3`; the handler's call site now defers to it and the text is unchanged byte for byte. **Regression test** `Tests/Spatial/TestGroundPlacement.cpp`, appended beside the existing `GroundProvenanceNamesTheMeasuredSurface` rather than filed as a parallel mechanism: `PinWright.spatial.verify_grounding.GroundProvenanceNamesTheAnsweringComponent`. Two candidate surfaces that differ ONLY in the component carrying them — a wide static-mesh floor at Z 50 and a one-instance HISM scatter at Z 250, both opaque, both blocking, both accepted by `any_solid` — with the prop resting on the scatter. Asserted in BOTH directions, because a report that named some component regardless of which one answered would pass a one-directional test: by default the block must name the HISM component, its class and its holder and must NOT name the floor's; then with `excludeComponentClasses` peeling the scatter off, the SAME prop in the SAME position must report the floor's `StaticMeshComponent` AND `maxGapCm` must move by the 200 cm between the two surfaces, so the name provably tracks the hit rather than being read off the actor or off whatever is in the level. Red before the change in the plainest way: `surfaceComponents` did not exist. **Not compiled and not run** — a full automation suite was running against the compiled DLL for the whole of this pass, per the wave's instruction. Docs: `Docs/wiki-src/spatial.ground-placement.md` gained a paragraph on `surfaceComponents` immediately after the existing `groundProvenance` one, stating what it is for (see a `componentClass` of `HierarchicalInstancedStaticMeshComponent` where you expected `LandscapeHeightfieldCollisionComponent` and the SURFACE SPEC, not the seat, is what needs fixing) and what its absence means. Concurrency: `Handlers/Environment/FoliageHandler.cpp` was edited in the same pass for the sibling ticket and nothing else in these files was touched; `B-ground-probe-hits-hull-not-render` cites the `ApplyPreset` `bTraceComplex` line, which this change again leaves byte-identical.
