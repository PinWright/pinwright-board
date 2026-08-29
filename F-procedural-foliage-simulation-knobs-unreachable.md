---
id: F-procedural-foliage-simulation-knobs-unreachable
title: "foliage.create_procedural writes four properties onto each generated UFoliageType and none of the fourteen that drive the procedural simulation — ProceduralScale, InitialSeedDensity, the clustering radii, the age/growth set and OverlapPriority are unreachable, and two of them interact in ways that make a caller's correct input read as a no-op"
status: OPEN
severity: High
category: feature
tags: [foliage, create_procedural, procedural-foliage-spawner, foliage-type, simulation, overlap-priority, max-initial-age, tile-size, hardcoded, missing-parameters, vegetation, ordering-constraint]
encounters: 1
lastSeen: 2026-08-29
---

# The verb builds the simulation's inputs and exposes almost none of them

`foliage.create_procedural`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1273`) creates a
`UProceduralFoliageSpawner` and one `UFoliageType_InstancedStaticMesh` per entry, then runs the
simulation. Everything the simulation reads off those assets is decided by the handler, and the
handler writes six properties in total:

- on the spawner — `TileSize` (`:1371`), `NumUniqueTiles` (`:1372`), `RandomSeed` (`:1373`);
- on each foliage type — `Density` (`:1447`), `ReapplyDensity` (`:1448`), and whatever
  `ApplyFoliageScaleAndAlign` writes (`:1449`), which is `Scaling`, `ScaleX/Y/Z.Min/Max` and
  `AlignToNormal` (`:117-129`).

**Fourteen `UFoliageType` properties that the procedural simulation reads are written nowhere in the
plugin.** Verified two ways: each name exists on the engine class, and each has **zero** occurrences
in the entire `Plugins/PinWright/Source` tree (all modules, tests included) — not merely in
`FoliageHandler.cpp`.

| property | engine declaration (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageType.h`) |
| --- | --- |
| `ProceduralScale` | `:503` — `FFloatInterval` |
| `InitialSeedDensity` | `:450` |
| `CollisionRadius` | `:436` |
| `ShadeRadius` | `:440` |
| `NumSteps` | `:444` |
| `SeedsPerStep` | `:462` |
| `AverageSpreadDistance` | `:454` |
| `MaxAge` | `:491` |
| `MaxInitialAge` | `:487` |
| `OverlapPriority` | `:499` |
| `bCanGrowInShade` | `:476` |
| `bSpawnsInShade` | `:483` |
| `RandomPitchAngle` | `:236` |
| `Height` | `:244` — `FFloatInterval` |
| `GroundSlopeAngle` | `:240` — `FFloatInterval` |

(Fifteen rows; `ProceduralScale` is the one that overlaps an existing ticket — see below — leaving
fourteen this ticket adds.)

Two further values are decided by the handler and settable by nobody:

- **`TileOverlap`, hardcoded to 0** — `ProcComp->TileOverlap = 0.0f;` at `FoliageHandler.cpp:1528`,
  against `UProceduralFoliageComponent::TileOverlap`
  (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/ProceduralFoliageComponent.h:52`). It is not
  echoed in the response (`:1580-1604` lists `tile_size` and `num_unique_tiles` and not this), so
  unlike the tiling pair a caller cannot even observe it.
- **The output path, hardcoded to `/Game/ProceduralFoliage`** — `FoliageHandler.cpp:1358`; both the
  spawner (`:1361-1364`) and every `_FT_<n>` type (`:1439-1441`) land there. The generated wiki
  *documents* this (`Saved/PinWright/wiki/foliage.create_procedural.md:21`), so it is honest, not
  silent — but every call writes into one shared folder with names derived from `name`, and there is
  no `savePath` the way `pcg.create_graph` has one.

## Do not re-litigate what already shipped

`B-create-procedural-ignores-scale-and-normal-fields`' `#2-honoured-scale-align-and-tiling` landed
`minScale` / `maxScale` / `alignToNormal` (per-type, via the shared
`ReadFoliageScaleAndAlign` / `ApplyFoliageScaleAndAlign` helpers) and `tileSize` / `numUniqueTiles`
(spawner-level, validated and echoed). **Those five are done and are not in this ask.** This ticket
is scoped to the remainder in the table above plus `TileOverlap` and the output path.

**One field overlaps, exactly one, and it is deliberate.** That ticket was reopened as
`#3-scale-write-lands-on-a-property-procedural-never-reads` because the scale write landed on
`ScaleX/Y/Z`, which `FPotentialInstance::PlaceInstance` reads only on the **non**-procedural branch;
the procedural branch takes `Settings->GetScaleForAge(...)`, which interpolates `ProceduralScale`.
So `ProceduralScale` appears both there and in the table above. **A fixer taking this ticket closes
that one**, provided the write goes *alongside* `ScaleX/Y/Z` rather than instead of it — `foliage.add_type`
shares the same helper and its painting path genuinely needs the paint-mode fields.

## The interaction constraints, which are the actual content of this ask

Each of these cost real debugging time on a live four-species vegetation level this session. A verb
that exposes these fields without stating them will produce callers who set them correctly and get
nothing.

**1. `OverlapPriority` must be ordered by effective radius, or the large species is annihilated.**
The engine's rule is *"When two instances overlap we must determine which instance to remove. The
instance with a lower OverlapPriority will be removed"* (`FoliageType.h:493-497`). It is not a
"which species do I like more" knob — it is evaluated against a competitor whose `CollisionRadius`
and `ShadeRadius` differ, so a small-radius species with equal or higher priority wins vastly more
overlap contests than its share of the area. Observed on this level: trees went **46 → 0** placed
when a ground-cover species was given equal priority, and **85 / 1 / 2 / 0** across four species
when the correction over-shot. Priority must be assigned in the same order as effective radius, and
a verb exposing it should say so at the parameter, not in a topic page.

**2. `MaxInitialAge` must be > 0, or every instance sits on a discrete scale ladder regardless of
the requested range — and this is why a `ProceduralScale` fix alone would look like it had not
worked.** The mechanism is arithmetic, not tuning:

- `GetInitAge` returns `MaxInitialAge * RandomStream.GetFraction()`
  (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:995-998`). Engine default
  `MaxInitialAge = 0` (`InstancedFoliage.cpp:655`), so **every seed starts at age exactly 0**.
- `GetNextAge` advances the age by integer 1 per step, capped at `MaxAge`
  (`InstancedFoliage.cpp:1000-1014`), over `NumSteps` steps — engine default `NumSteps = 3`
  (`InstancedFoliage.cpp:649`).
- `GetScaleForAge` evaluates the scale curve at `Age / MaxAge` and maps it into the interval
  (`InstancedFoliage.cpp:987-993`).

Ages therefore take four values — 0, 1, 2, 3 — and the scale takes four values, whatever
`ProceduralScale` is set to. `B-create-procedural-ignores-scale-and-normal-fields`' `#3` measured
exactly that on placed instances: `{1.0, 1.2, 1.4, 1.6}`. `MaxInitialAge` is what turns the ladder
continuous, so it is not an optional extra alongside `ProceduralScale` — it is the field that makes
`ProceduralScale` observable at all.

**3. `Height` and `GroundSlopeAngle` are placement FILTERS, and the tile simulation is terrain-free
— so they cannot give species elevation niches.** Both are `Category=Placement` and both are
documented as rejection ranges: *"The valid altitude range where foliage instances will be placed,
specified using minimum and maximum world coordinate Z values"* (`FoliageType.h:242-244`) and
*"Foliage instances will only be placed on surfaces sloping in the specified angle range from the
horizontal"* (`:238-240`). The simulation runs on a flat abstract tile and the results are then
projected; the seeds are not competing *on* terrain. A caller who sets `Height` expecting alpine
species above a treeline and riparian species below gets thinning, not stratification — the excluded
seeds are simply discarded, so the high band comes out sparse rather than differently populated.
**Any verb exposing these two must document them as filters over an already-simulated set**, or it
will be read as the stratification control it looks like.

**4. The plugin's `tileSize` default is 10x smaller than the engine's, and that — not `randomYaw` —
is what produces visible cloned tiling.** Plugin default is 1000 (`FoliageHandler.cpp:1340`); the
engine's `UProceduralFoliageSpawner` constructor sets `TileSize = 10000` with the comment `//100 m`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/ProceduralFoliageSpawner.cpp:17`).
`NumUniqueTiles` agrees at 10 (`:19` vs `FoliageHandler.cpp:1345`). Ten unique 10 m tiles repeated
across a large volume is a visible repeat at close range; ten unique 100 m tiles is not. This one is
already settable (`#2` above landed it) and is recorded here because the *default* is the thing
callers hit, and because `randomYaw` — which this verb genuinely does not read
(`FoliageHandler.cpp:1403`, the surviving `UnreadPerTypeFields` entry) — is the wrong suspect and
the one a caller reaches for first.

## What the ask actually is

Not "expose fourteen fields". Expose them **with the ordering and interaction constraints stated at
the parameter**, in a nested per-type schema alongside the `minScale` / `maxScale` / `alignToNormal`
that already live there, plus `tileOverlap` and a `savePath` on the call. Concretely, the four
constraints above become four documented rules a caller can follow:

- `overlapPriority` — doc states the radius-ordering requirement and that a mis-order silently
  wipes a species; the verb is well placed to *validate* it, since it holds every entry's
  `collisionRadius` / `shadeRadius` in the same call and can refuse or warn on a priority order that
  inverts the radius order. That validation is the single highest-value thing in this ticket.
- `maxInitialAge` — doc states that leaving it at 0 collapses `proceduralScale` to `numSteps + 1`
  discrete values.
- `height` / `groundSlopeAngle` — doc states filter-not-niche.
- `tileSize` — reconsider the 1000 default against the engine's 10000, or document the divergence
  where a caller reads it. (Changing it is a behaviour change for existing callers; `#2`
  deliberately left it alone. Stating it costs nothing.)

Follow the shape `#2` established: per-type keys read through the shared
`ReadFoliageScaleAndAlign`-style helper so `foliage.add_type` and `foliage.create_procedural` cannot
drift, a bad value routed through the existing `NoteSkippedType` channel with a reason rather than
failing the batch (`FoliageHandler.cpp:1383-1391`), and the effective values echoed.

**Severity: High, argued.** The bulk of the fourteen are the rubric's *"High or Medium: hard blocker
with no workaround"* — the simulation properties are the simulation, and a caller cannot reach them:
`property.set` on the generated `_FT_<n>` assets writes the field, but nothing re-runs the
simulation to consume it, because no verb re-simulates an existing volume (see
`F-resimulate-existing-foliage-volume`, filed alongside this one). The chain is broken at the far
end, so the obvious workaround does not close. What decides High over Medium is that this ticket
**contains** a field already rated High on another ticket: `B-create-procedural-ignores-scale-and-normal-fields`
`#3` raised that ticket to High on `ProceduralScale` specifically — silent wrong data on a normal
path, the caller told twice that a scale range was honoured when the simulation never reads it — and
argued it against this same rubric. Rating this Medium would sort the ticket that *closes* that
defect below the ticket that reports it, which is exactly the mis-ordering the severity field
exists to prevent. **Reach modifier declined in both directions, and named:** `foliage.create_procedural`
is one of six verbs in its namespace — not an every-session method that would earn the bump up, and
not a rare edge path, since it is the plugin's entire procedural-vegetation surface. High stands
unmodified. Not Critical: nothing is corrupted or lost; the scatter is merely wrong, and the assets
it writes are re-creatable.

## Related

- `B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High, `encounters: 2`) — reopened at
  `#3` because the scale write landed on `ScaleX/Y/Z`, which the procedural path never reads.
  **Overlaps this ticket on exactly one field, `ProceduralScale`**; a fixer taking this one closes
  that one. Its `#2` is also the template for how per-type nested keys should be added here.
- `B-create-procedural-density-writes-paint-density` — same verb, same shape: `density` writes
  `UFoliageType::Density` (`FoliageHandler.cpp:1447`) while the simulation reads
  `InitialSeedDensity` (`FoliageType.h:450`), which is also row 2 of the table above. Same overlap
  relationship as `ProceduralScale`: closing this closes that. *(Cross-linked from
  `B-create-procedural-ignores-scale-and-normal-fields` `#3`; not yet present in this working tree
  at the time of filing — being filed concurrently this session.)*
- `E-foliage-nested-input-schemas-undocumented` (IN-REVIEW, Low) — documents the nested
  `foliageTypes[]` / `bounds` shapes. Its field list and
  `Tests/Infra/TestFoliageNestedInputSchemaDocs.cpp` **must be updated in the same commit** as any
  change here; `#2` of the scale ticket records that requirement and the reason (the assertion
  pinned the old "not honored" doc text and would have kept passing against text that had become
  false).
- `F-resimulate-existing-foliage-volume` — why `property.set` on the generated assets is not a
  workaround for this ticket.
- `B-spawned-volumes-have-no-brush-geometry` (OPEN, High) — same verb: the
  `AProceduralFoliageVolume` it spawns has bounds extent `(0,0,0)`, which is why calls report
  `instances_spawned: 0`. Anyone testing a fix to this ticket must fix or work around that first, or
  every measurement will read zero for an unrelated reason.
- `F-foliage-namespace-has-no-behavioural-tests` — `create_procedural`'s coverage gap, which is why
  none of this was caught by the suite.

## History
- `#1-fourteen-simulation-knobs-unreachable` `OPEN` reporter — Verified every property name against
  the engine class (`FoliageType.h` lines in the table above;
  `ProceduralFoliageComponent.h:52` for `TileOverlap`;
  `ProceduralFoliageSpawner.h:21-27` for the spawner fields) and confirmed **zero** occurrences of
  all fifteen names anywhere under `Plugins/PinWright/Source` — not just in `FoliageHandler.cpp`.
  Enumerated what the handler does write: spawner `TileSize`/`NumUniqueTiles`/`RandomSeed`
  (`:1371-1373`), per-type `Density`/`ReapplyDensity` (`:1447-1448`) and
  `ApplyFoliageScaleAndAlign`'s `Scaling`/`ScaleX/Y/Z`/`AlignToNormal` (`:117-129`, called at
  `:1449`). Hardcodes cited: `TileOverlap = 0.0f` (`:1528`, not echoed at `:1580-1604`) and the
  output path `/Game/ProceduralFoliage` (`:1358`, documented at
  `Saved/PinWright/wiki/foliage.create_procedural.md:21` but not settable). Scoped away from
  `B-create-procedural-ignores-scale-and-normal-fields` `#2`, which already landed `minScale` /
  `maxScale` / `alignToNormal` / `tileSize` / `numUniqueTiles`; `ProceduralScale` is the single
  deliberate overlap with that ticket's reopened `#3`. The `MaxInitialAge` constraint was verified
  arithmetically rather than accepted: `GetInitAge` = `MaxInitialAge * GetFraction()`
  (`InstancedFoliage.cpp:995-998`) with engine default 0 (`:655`), `GetNextAge` stepping integer ages
  capped at `MaxAge` (`:1000-1014`) over `NumSteps` default 3 (`:649`), and `GetScaleForAge`
  (`:987-993`) mapping those four ages into the interval — the `{1.0, 1.2, 1.4, 1.6}` ladder that
  ticket measured on placed instances. `Height` / `GroundSlopeAngle` confirmed `Category=Placement`
  rejection ranges from their own doc comments (`FoliageType.h:238-244`). `tileSize` divergence
  confirmed against `ProceduralFoliageSpawner.cpp:17` (`TileSize = 10000; //100 m`) versus the
  plugin's 1000 at `FoliageHandler.cpp:1340`. The `OverlapPriority` radius-ordering rule is field
  evidence from this session (trees 46 → 0, then 85/1/2/0 over-corrected) read against the engine's
  own semantics at `FoliageType.h:493-499`.
