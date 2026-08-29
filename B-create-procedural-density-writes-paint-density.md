---
id: B-create-procedural-density-writes-paint-density
title: "foliage.create_procedural's per-type density writes UFoliageType::Density — the PAINT-BRUSH knob — while the procedural simulation seeds from InitialSeedDensity, which the handler never writes, so every generated type simulates at the CDO default of 1 seed per 10 m tile no matter what the caller asked for"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, create_procedural, procedural-foliage-spawner, density, initial-seed-density, inert-write, silent-wrong-data, wrong-property, vegetation, response-honesty]
---

# The one per-type field this verb claims to read lands on a property the simulation never opens

`foliage.create_procedural` declares its per-type payload as *"Array of foliage type configs with
meshPath, density, minScale, maxScale and alignToNormal"*
(`Handlers/Environment/FoliageHandler.cpp:1277`). It reads `density` per entry at
`FoliageHandler.cpp:1416-1417`:

```cpp
double TypeDensity = 10.0;
(*TypeObj)->TryGetNumberField(TEXT("density"), TypeDensity);
```

and writes it at `:1447-1448`:

```cpp
FT->Density = (float)TypeDensity;
FT->ReapplyDensity = true;
```

`UFoliageType::Density` is declared under `// PAINTING`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageType.h:150-152`), category `Painting`,
display name `"Density / 1Kuu"`, commented *"Foliage instances will be placed at this density,
specified in instances per 1000x1000 unit area"*. It is the interactive Foliage-editor brush's
knob. `ReapplyDensity` (`:154-156`) is its reapply-tool companion. Neither is procedural.

## The field the simulation actually reads

`UFoliageType::InitialSeedDensity` — declared under `// CLUSTERING`
(`FoliageType.h:448-450`), category `Procedural`, subcategory `Clustering`, commented *"Specifies
the number of seeds to populate along 10 meters. The number is implicitly squared to cover a 10m x
10m area"*. It is exposed only through `GetSeedDensitySquared()` (`FoliageType.h:425`), and that
accessor has exactly one caller in the whole engine — the tile simulation:

```cpp
const int32 NumSeeds = FMath::RoundToInt(TypeInstance->GetSeedDensitySquared() * SizeTenM2);
SeedsLeftMap.Add(TypeInstance, NumSeeds);
if (NumSeeds > 0)
{
    TypesToSeed.Add(TypeInstance);
}
```

`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/ProceduralFoliageTile.cpp:231-235`, inside
`FProceduralFoliageTile::Simulate`'s seeding loop, where `SizeTenM2` is
`(FoliageSpawner->TileSize * FoliageSpawner->TileSize) / (1000.f * 1000.f)` (`:215`). That is the
whole of the caller's control over how many plants a procedural pass produces.

**`InitialSeedDensity` appears nowhere in the plugin.** `grep -rn "InitialSeedDensity"` over
`Plugins/PinWright/Source/` returns nothing — not in `FoliageHandler.cpp`, not in
`ApplyFoliageScaleAndAlign` (`:117-130`, the shared writer both foliage verbs use), not anywhere
else. (It matches three `Intermediate/` `.obj` files and one `.pdb`; that is the engine header
inlined into the build, not plugin code.) So every type this verb builds keeps the CDO value,
`InitialSeedDensity = 1.f` (`InstancedFoliage.cpp:652`).

**What that means numerically.** With the verb's default `tileSize` of 1000
(`FoliageHandler.cpp:1273-1281` registration, read at `:1340`, written at `:1371`), `SizeTenM2` is
`1000*1000/1000000` = 1, so `NumSeeds = round(1.0² × 1)` = **1 initial seed per tile, per type** —
whatever `density` the caller passed. The number is not scaled, clamped or approximated by the
requested value; the requested value is not in the computation at all.

## Why this is silent rather than merely wrong

`density` is not in `UnreadPerTypeFields` (`FoliageHandler.cpp:1401`), the array whose members get
echoed into `ignoredFields` (`:1409-1411`, emitted at `:1595-1601`). That array currently holds one
name, `randomYaw`. So the response says nothing about `density` — and by the file's own stated
contract (`:1379-1380`: *"Per-entry keys this verb does NOT read are echoed back in `ignoredFields`
so a caller who set them learns they had no effect"*) a key's absence from `ignoredFields` is a
positive claim that the key was read. It was read. It was written. It landed on the wrong property.

The second confirmation is worse: the write is real, so a caller who does the responsible thing and
inspects the generated `_FT_<n>` asset gets their own number back. `property.get` on
`UFoliageType_InstancedStaticMesh::Density` returns exactly what they asked for, while the scatter
that asset produced was seeded from a `1.0` they never saw. Both the response and the readback
agree, and both are describing a field nothing downstream reads.

## This contradicts the ticket it is cross-linked from

`B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High) is titled

> "foliage.create_procedural builds each UFoliageType with only mesh and density"

and frames its finding as *"Two classes of input never reach the assets it builds"*, with `density`
outside both classes — the one thing that was supposed to work. Its severity argument leans on the
same assumption:

> "A forest where every tree is exactly the same size and every trunk is exactly vertical reads as
> wrong at a glance, and **no amount of density tuning fixes it**."

There is no density tuning to run out of. The knob was never connected.

**The consequence for a triager is that the count is wrong too, not only the appearance.** Read
together with that ticket's own `#3` — which returned the scale half because `ScaleX/Y/Z` is read
only on the non-procedural branch of `FPotentialInstance::PlaceInstance` and the procedural branch
takes `GetScaleForAge`/`ProceduralScale` instead — **every declared per-type field on
`foliage.create_procedural` is now known to be inert on the procedural path**: `minScale`/`maxScale`
land on `ScaleX/Y/Z`, `density` lands on `Density`, and the simulation reads `ProceduralScale` and
`InitialSeedDensity`. `alignToNormal` is the sole survivor (verified live in that ticket's `#3`,
tilting to 37.7° on slope). The pattern is not "some fields were forgotten"; it is that the handler
writes the **painting** half of `UFoliageType` throughout and the verb drives the **procedural**
half.

That ticket's `#3` already names this defect in one clause and reserves this id. This ticket exists
because the claim it contradicts is in that ticket's *title and body*, which is what a triager reads
before the history, and because the fix here is a different line in a different property block.

## Fix

Write `InitialSeedDensity` from the same `density` input, in `ApplyFoliageScaleAndAlign`
(`FoliageHandler.cpp:117-130`) alongside the scale fields, **as well as** `Density` — not instead of
it. Both foliage verbs share that helper and `foliage.add_type`'s painting path genuinely needs
`Density`, so the helper should set both and the two call sites (`:957`, `:1449`) stay identical.
That is the same shape the sibling ticket's `#3` prescribes for `ProceduralScale`, and the two fixes
should land together or the second will look like a regression of the first.

Two things a fixer must not assume:

- **`density` and `InitialSeedDensity` are not the same unit.** `Density` is instances per
  1000×1000 uu; `InitialSeedDensity` is seeds along 10 m, *implicitly squared* to cover 10 m × 10 m
  (`FoliageType.h:447`). A raw copy makes `density: 100` mean 10 000 seeds per tile. Either convert
  (`sqrt`) with the conversion stated in the param description and echoed in the response, or —
  cleaner — take a separate `seedDensity` per type and stop overloading one name for two units.
  Whichever is chosen, say which one the number means, in the registration string, because the
  current one says only *"density"*.
- **Verify on placed instances, not on the asset.** Asserting `InitialSeedDensity` off the generated
  `_FT_<n>` is exactly the assertion that let this through: the existing regression test
  `PinWright.foliage.create_procedural.AppliesScaleAlignAndTiling` reads the property the simulation
  ignores and is green. The test that would have caught this compares `instances_spawned` (`:1603`,
  a measured before/after delta) across two runs at different densities on the same bounds.

Also worth doing in the same pass: if a per-type key is read and written but demonstrably inert
downstream, `ignoredFields` is the wrong shape to report it — the key *was* read. A sibling
`inertFields` (or a `warnings[]` line naming the property that was written and the property the
simulation reads) is what closes the honesty gap for any knob that cannot be wired straight away.

## Same shape as

`B-foliage-paint-does-no-ground-projection` carries the fullest statement of the class: *the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported.* This is that shape with an extra turn — the deciding number was not merely
unreported, it was accepted, echoed as read, and stored somewhere nothing looks.

`B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High) — same verb, same property
block, same wrong-half-of-the-UClass mechanism; that ticket's `#3` is the scale instance of this
one's density instance.

`F-procedural-foliage-simulation-knobs-unreachable` — reserved and being filed in this session for
the ~14 further `Category=Procedural` properties (`CollisionRadius`, `ShadeRadius`, `NumSteps`,
`AverageSpreadDistance`, `MaxInitialAge`, `ProceduralScale`, `MinimumQuadTreeSize`, …) with no path
through this verb at all. **The split is deliberate and a triager should keep it:** those knobs were
never advertised, so a caller was never told anything false about them — that is a missing feature.
`density` *is* advertised, accepted, and reported-as-read, which is why it is a bug. If that feature
ticket is taken first and adds a general per-type property route, this one closes with it.

`B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW, High) — same verb, the resimulation
callback rather than the type build; its fix is what makes `instances_spawned` a measured delta and
therefore what makes this defect observable at all.

## Not RPC-verified

Source-read only; the editor was not running for this pass. No `foliage.create_procedural` call was
made and no generated `_FT_<n>` asset was inspected. The claim that the requested `density` never
enters the seed count is an inference from the engine call graph — `GetSeedDensitySquared()` has one
caller, `ProceduralFoliageTile.cpp:231`, and `Density` has none on that path — not an observation of
a scatter. The measurement that would settle it is two `create_procedural` calls on identical bounds
with `density: 10` and `density: 1000`, comparing `instances_spawned`; this ticket predicts the two
counts are equal.

severity rationale: impact=High — silent wrong data on a normal path, in the rubric's exact sense: a declared parameter is accepted, is deliberately excluded from the `ignoredFields` mechanism that the handler's own comment (`:1379-1380`) defines as the signal for an unread key, is written to a real property, and reads back correct from the generated asset — so the caller is told the same false thing twice and builds a vegetation pass on it, while the simulation seeds from a CDO `1.0` (`InstancedFoliage.cpp:652`) that appears nowhere in the response; this is the same argument on which the sibling ticket's `#3` raised the scale half Medium -> High, and rating this one lower would leave two identical defects in the same loop at different bands. Weighed against High and rejected: the response does not *state* a resulting density anywhere, so no numeric field in it is literally false — but the rubric's High band is about a caller trusting a result that is a lie, and "your `density` was read" is the claim being made, by the file's own stated contract; and a caller cannot distinguish a sparse scatter from a requested one, which is the mechanism of the silence rather than a mitigation of it. Not Critical: nothing is corrupted or lost, the generated assets are valid and re-simulating after a manual `InitialSeedDensity` edit recovers the intended scatter × reach=normal — `foliage.create_procedural` is one of six verbs in the foliage namespace, not an every-session method and not a rare edge path, so the bump-down is declined; the reading that would take it to Medium is reach-as-observed-usage (this namespace has no recorded use on this project), and that is an `encounters` signal the rubric excludes from severity outright -> High

## History
- `#1-density-lands-on-the-painting-knob` `OPEN` reporter — Source-read only, editor not running; no `foliage.create_procedural` call was made and no generated `_FT_<n>` was inspected. Every citation below re-derived against this tree, not inherited. `density` is declared in the `foliageTypes` param description (`Handlers/Environment/FoliageHandler.cpp:1277`), read per entry with a default of `10.0` (`:1416-1417`), and written as `FT->Density = (float)TypeDensity; FT->ReapplyDensity = true;` (`:1447-1448`). `UFoliageType::Density` is under `// PAINTING`, `Category=Painting`, `DisplayName="Density / 1Kuu"` (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageType.h:150-152`) — the interactive brush knob. The procedural simulation instead seeds from `InitialSeedDensity` (`FoliageType.h:448-450`, `Category=Procedural`, `Subcategory="Clustering"`), reachable only via `GetSeedDensitySquared()` (`:425`), whose sole engine caller is `FProceduralFoliageTile::Simulate` — `const int32 NumSeeds = FMath::RoundToInt(TypeInstance->GetSeedDensitySquared() * SizeTenM2);` (`Runtime/Foliage/Private/ProceduralFoliageTile.cpp:231`, `SizeTenM2` at `:215`). NEGATIVE PROVEN: `grep -rn "InitialSeedDensity" Plugins/PinWright/Source/` returns nothing — the only matches under the plugin are three `Intermediate/` `.obj` files and the `.pdb`, i.e. the inlined engine header, not plugin code. So every generated type keeps the CDO `InitialSeedDensity = 1.f` (`InstancedFoliage.cpp:652`), and at the verb's default `tileSize` 1000 (`:1340`, `:1371`) that is `NumSeeds = round(1.0^2 * 1)` = 1 seed per tile per type regardless of the request. Silence is structural, not incidental: `density` is absent from `UnreadPerTypeFields` (`:1401`, currently `{randomYaw}` only), the array echoed as `ignoredFields` (`:1409-1411`, emitted `:1595-1601`), and the handler's own comment at `:1379-1380` defines that absence as meaning the key WAS read; a `property.get` on the generated asset then returns the caller's own number, so response and readback agree on a field nothing downstream opens. CONTRADICTS THE CROSS-LINKED TICKET, which is why this was filed rather than appended: `B-create-procedural-ignores-scale-and-normal-fields` (OPEN, High) is titled "builds each `UFoliageType` with only mesh and density" and argues severity from "no amount of density tuning fixes it" — both presume a working density knob. That ticket's `#3` already reserves this id and names the defect in one clause; the contradicted claims are in its title and body, which is what a triager reads first, and its file was not edited. Also relevant to that ticket: several of its body citations have drifted ~350 lines (`:1037-1039` for `UnreadPerTypeFields` is now `:1401`; `:1074-1076` for the three-line type build is now `:1445-1448`; `:935-939` for the param list is now `:1273-1281`), and the `minScale`/`maxScale`/`alignToNormal` half it describes as unwritten IS now written (`ApplyFoliageScaleAndAlign`, `:117-130`, called at `:1449`) — its `#3` returned it because the procedural branch reads different properties, not because the write is missing. Dedup: grepped the board for `InitialSeedDensity`, `paint density`, `UFoliageType::Density`, `create_procedural`, `ProceduralFoliageSpawner` and every `B-foliage-*` / `E-foliage-*` / `*-create-procedural-*` file. Only `B-create-procedural-ignores-scale-and-normal-fields` mentions `InitialSeedDensity` at all, and only in its `#3`'s reservation of this id. `B-create-procedural-terrain-paints-nothing` is `environment.create_procedural_terrain`, a different verb. `E-create-procedural-mesh-sparse-response` is `geometry`. `B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW) is the same verb's resimulation callback, a different mechanism. Nothing on the board covers this. Recorded, not filed here: `F-procedural-foliage-simulation-knobs-unreachable` is being filed in this session for the ~14 further `Category=Procedural` properties that have no path through this verb at all — deliberately split from this ticket because those were never advertised (a missing feature) whereas `density` is advertised, accepted and reported-as-read (a defect).
- `#2-density-derives-initial-seed-density` `IN-REVIEW` developer — Taken in one pass with `F-procedural-foliage-simulation-knobs-unreachable` and `B-create-procedural-ignores-scale-and-normal-fields` `#3`, because all three rewrite the same per-type property block. **The Fix section's position followed, with one deliberate deviation on placement.** `density` still writes `UFoliageType::Density` — a caller who later paints with the generated `_FT_<n>` needs it, and `foliage.add_type` writes the same thing — and `InitialSeedDensity` is now written **as well**, derived from the same number. The conversion the ticket insisted on is applied and stated at the parameter: `density` is instances per 1000x1000 uu, `InitialSeedDensity` is seeds along 10 m *implicitly squared* over 10 m x 10 m, so the derivation is `sqrt(density)` and a raw copy would have turned `density: 100` into 10 000 seeds a tile. The ticket's cleaner alternative is also there: a new per-type `initialSeedDensity` key, taken **verbatim** when supplied, so a caller who wants the engine unit directly never goes through the conversion. **COMPATIBILITY CONSEQUENCE, stated because it is real.** Existing callers' scatters change. Before this, every generated type simulated at the CDO `InitialSeedDensity = 1.0` regardless of `density` — one seed per tile at the verb's default `tileSize` 1000. Now the verb's own default `density` 10 derives 3.162, and a caller who passed `density: 300` gets 17.3, i.e. roughly 300 seeds per 100 m2 rather than 1. That is the fix working, but a caller who tuned other knobs around a dead one will see a denser result than before. It is not silent: the new per-type `foliage_types[]` echo carries `initial_seed_density` and `initial_seed_density_source` (`"supplied"` | `"derived_from_density"`) on every call, so the number that decided the scatter is in the response for the first time. **Deviation, argued rather than assumed.** The Fix says to put the write in `ApplyFoliageScaleAndAlign` so the two verbs' call sites stay identical. It is instead in a new `create_procedural`-only helper pair (`ReadFoliageProceduralSimConfig` / `ApplyFoliageProceduralSimConfig`, file scope, next to the scale helpers). The reason the scale helper is shared is that BOTH verbs accept `minScale`/`maxScale`/`alignToNormal` and could drift on them; `foliage.add_type` accepts none of the fifteen simulation keys and runs no simulation, so there is no shared surface to drift on, and its `density` genuinely is the paint brush's. Following the letter would also have meant moving the `Density` write itself into the shared helper (it is currently at each call site), restructuring a painting verb for a procedural reason. The comment above the new helpers states this so the next reader does not re-open it. **The `inertFields` suggestion was not needed and was not added**: `density` is no longer inert. What grew instead is `UnreadPerTypeFields`, which now also carries `spreadVariance`, `distributionSeed`, `maxInitialSeedOffset`, `scaleCurve` and `minimumQuadTreeSize` — the properties still unreachable — so a caller who guesses one of those engine names is told in `ignoredFields` rather than having it dropped. **Files changed:** `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, `Docs/wiki-src/foliage.md` (create_procedural H3: the simulation-key table, the four interaction rules, the `foliage_types[]` readback, the still-unreachable list, worked example), `Source/PinWright/Private/Tests/Infra/TestFoliageNestedInputSchemaDocs.cpp` (one more overlay-exclusive marker pair — `initial_seed_density_source` and `spreadVariance` — per `E-foliage-nested-input-schemas-undocumented`'s same-commit requirement). **Regression test:** `Source/PinWright/Private/Tests/World/TestFoliageCreateProceduralSimulationConfig.cpp`, `PinWright.foliage.create_procedural.AppliesSimulationConfig` — routed through `FRpcDispatcher::ProcessRequest`, it opens the generated `_FT_0` and asserts `InitialSeedDensity == 10` (`sqrt(100)`), not the CDO 1.0, while `Density` still reads the requested 100; a second entry supplies `initialSeedDensity: 7` alongside `density: 400` and asserts 7 verbatim, i.e. that supplied beats derived. Pre-fix the run stops at the dispatcher's `UNKNOWN_PARAMS` (`tileOverlap` undeclared) and, past that, every readback would be a CDO value. **The ticket's "verify on placed instances, not on the asset" instruction is acknowledged and NOT satisfied, deliberately.** `B-spawned-volumes-have-no-brush-geometry` is why: it makes `instances_spawned` read zero for an unrelated reason, so a two-density instance-count comparison is not a usable suite assertion until that lands. The asset readback is discriminating here in a way `#3`'s critique of the sibling test was not, because the property being read is now the one `FProceduralFoliageTile::Simulate` opens rather than the one it ignores — but a live two-call density comparison is still the measurement that would settle it, and it has not been run. Not compiled and not run — per instruction, the wave owner builds and runs the suite.
