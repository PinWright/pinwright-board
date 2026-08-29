---
id: B-create-procedural-ignores-scale-and-normal-fields
title: "foliage.create_procedural builds each UFoliageType with only mesh and density — it drops per-type minScale/maxScale/alignToNormal (honestly, into ignoredFields) and hardcodes the spawner's TileSize and NumUniqueTiles (silently, with no echo at all), so every plant is the same size, upright, and tiled on a 10 m grid"
status: OPEN
severity: High
category: bug
tags: [foliage, create_procedural, procedural-foliage-spawner, scale, align-to-normal, tile-size, hardcoded, ignored-fields, response-honesty, vegetation]
encounters: 2
lastSeen: 2026-08-29T00:00:00+05:00
---

# The spawner's own variation controls are the ones that never arrive

`foliage.create_procedural` takes `name`, `bounds`, `foliageTypes[]` and `seed`
(`Handlers/Environment/FoliageHandler.cpp:935-939`). It creates a
`UProceduralFoliageSpawner`, one `UFoliageType_InstancedStaticMesh` per entry, a volume, and runs
the simulation. Two classes of input never reach the assets it builds, and the two are reported
very differently.

## Half of it is honest, and that echo is the evidence

Per-type `minScale`, `maxScale` and `alignToNormal` are collected and returned:

```cpp
// Read by foliage.add_type, NOT by this verb - see the create_procedural overlay.
static const TCHAR *const UnreadPerTypeFields[] = {
    TEXT("minScale"), TEXT("maxScale"), TEXT("alignToNormal")};
```

`:1037-1039`; gathered per entry at `:1046-1051`; emitted as `ignoredFields` at `:1216-1223`, and
only when non-empty. Nothing is being hidden — a caller who sets them is told they had no effect.
**That echo is what makes this ticket cheap to substantiate**: the code already names its own gap.
It is filed as a bug rather than an ergonomic because what it names is a dropped input on a write
verb, not a doc or discoverability problem.

What the type actually gets is three lines:

```cpp
FT->SetStaticMesh(Mesh);
FT->Density = (float)TypeDensity;
FT->ReapplyDensity = true;
```

`:1074-1076`.

**The sibling verb, in the same file, on the same UClass, already writes the missing fields.**
`foliage.add_type` sets `ScaleX.Min/Max`, `ScaleY.Min/Max`, `ScaleZ.Min/Max` (`:612-617`) and
`AlignToNormal` (`:618`) on a `UFoliageType_InstancedStaticMesh`. There is no engine obstacle and no
new API to find; the fix is to apply the same writes 460 lines further down. That is the whole
argument for treating this as a defect rather than a feature request.

Why it matters more here than the field list suggests: scale variation and normal alignment are
most of what a *procedural* spawner is for in vegetation work. A forest where every tree is exactly
the same size and every trunk is exactly vertical reads as wrong at a glance, and no amount of
density tuning fixes it. The verb produces a scatter that is correct in count and wrong in
appearance, and reports success.

## The other half is silent, and that is the actual defect

```cpp
Spawner->TileSize = 1000.0f; // Default tile size
Spawner->NumUniqueTiles = 10;
Spawner->RandomSeed = Seed;
```

`:1013-1015`. `seed` is a declared parameter and is honoured. The other two are **hardcoded, are not
parameters, and appear in no response field** — they cannot even reach `ignoredFields`, because that
array only reports keys the caller supplied and this verb never accepts them.

Engine semantics, so the numbers mean something
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/ProceduralFoliageSpawner.h:25-31`):
`TileSize` is *"Length of the tile (in cm) along one axis"* and `NumUniqueTiles` is *"The number of
unique tiles to generate. The final simulation is a procedurally determined combination of the
various unique tiles."* So every call simulates ten 10 m x 10 m tiles and repeats that set across
whatever `bounds` was given. On a volume of a few hundred metres that is a ten-tile pattern
repeating dozens of times — visible tiling in the finished scatter — and on a volume smaller than
10 m it is a single tile, i.e. no procedural variation at all. `MinimumQuadTreeSize` (`:35`) is left
at its CDO value and is likewise unreachable.

This is the part that fits the board's recurring shape: the call succeeds,
`foliage_types_count` / `instances_spawned` / `skippedCount` are all correct, and the deciding
parameters were never reported because they were never inputs.

## Fix

1. **Apply the three per-type fields.** In the type-construction block at `:1069-1076`, set
   `ScaleX/Y/Z` from `minScale`/`maxScale` and `AlignToNormal` from `alignToNormal`, reusing
   `foliage.add_type`'s validation (`:532-540`: positive scales, `minScale <= maxScale`) rather than
   duplicating it — promote it to a file-scope helper so the two verbs cannot drift. Then remove
   those three entries from `UnreadPerTypeFields` (`:1038-1039`) **in the same commit**, or the
   response will start lying in the opposite direction.
2. **Accept `tileSize` and `numUniqueTiles`** as optional parameters, and echo the effective values
   in the response whether defaulted or supplied — a spawner whose tiling the caller cannot see is
   the same defect one level up. Consider deriving a default `TileSize` from `bounds` rather than a
   constant, so the common case stops tiling visibly; if a constant is kept, echo it and say it is
   a default.
3. **Keep `ignoredFields` for whatever remains unread.** It is the right mechanism and the reason
   this ticket exists; do not delete it when the current three entries go.

## Distinct from

- **`B-foliage-create-procedural-empty-callback-noop`** (IN-REVIEW, High) — same verb, different
  defect: the instances were discarded by an empty `AddInstancesFunc`. **Confirmed fixed in this
  tree**: the reflection path through
  `/Script/FoliageEdit.ProceduralFoliageEditorLibrary::ResimulateProceduralFoliageComponents` is
  present at `:1176-1180`, and `instances_spawned` is now a measured before/after delta
  (`:1202-1203`, reported at `:1225`) rather than a hardcoded flag. That fix is what makes this ticket observable at all
  — you cannot notice that every plant is the same size until plants appear.
- **`E-foliage-nested-input-schemas-undocumented`** (IN-REVIEW, Low) — documents the nested shapes
  of `bounds` and `foliageTypes[]`. It describes what the payload looks like; this is about which
  parts of it are read. A fixer landing this ticket must update that doc's field list in the same
  commit.
- **`E-foliage-add-type-auto-save-undocumented`** (WONTFIX, Low) — `add_type` persistence, unrelated
  except that `add_type` is the verb whose scale/normal writes should be reused here.

## Not RPC-verified

Source-read only; the editor was not running for this pass. No `foliage.create_procedural` call was
made and no generated scatter was rendered. Two things an editor test would settle: whether the
simulation honours a `UFoliageType`'s `ScaleX/Y/Z` at all (it should — the procedural path builds
`FDesiredFoliageInstance`s the same `FPotentialInstance::PlaceInstance` consumes,
`Runtime/Foliage/Private/InstancedFoliage.cpp:5506`, but its `PlacementMode == Procedural` branch
takes a different rotation path at `:5540-5543`, so `alignToNormal` specifically may behave
differently under procedural placement than under painting), and what a realistic `TileSize` for a
given `bounds` is. Neither changes the fix; the second changes what the default should be.

severity rationale: impact=Medium — soft blocker, not a silent lie on the honest half: the three per-type fields are dropped but reported in `ignoredFields`, and a caller can work around it by authoring types with `foliage.add_type` first, so the caller is neither deceived nor stuck; the hardcoded `TileSize` / `NumUniqueTiles` half IS silent and has no workaround at all through this surface, which argues High, and it is declined here only because those two are absent parameters rather than accepted-and-discarded ones — the caller was never told they existed, so nothing they were told is false × reach=normal — `foliage.create_procedural` is one of six verbs in the namespace, not a rare edge path, so no modifier applies -> Medium

## History
- `#1-drops-scale-and-tiling` `OPEN` reporter — Source-read only, editor not running; no `foliage.create_procedural` call was made and no scatter was rendered. Per-type `minScale` / `maxScale` / `alignToNormal` are collected into `UnreadPerTypeFields` (`FoliageHandler.cpp:1037-1039`), matched per entry (`:1046-1051`) and echoed as `ignoredFields` (`:1216-1223`) — the verb is honest about dropping them, and that echo is the evidence for this ticket. The type it builds gets only `SetStaticMesh` / `Density` / `ReapplyDensity` (`:1074-1076`), while `foliage.add_type` in the same file sets `ScaleX/Y/Z` (`:612-617`) and `AlignToNormal` (`:618`) on the same `UFoliageType_InstancedStaticMesh` class, so there is no engine obstacle to applying them here. Separately and less honestly, `Spawner->TileSize = 1000.0f` and `Spawner->NumUniqueTiles = 10` are hardcoded (`:1013-1014`) with no parameter and no response field — unlike `RandomSeed` (`:1015`), which is a declared param. Per `ProceduralFoliageSpawner.h:25-31` that means every call simulates ten 10 m tiles and repeats them across `bounds`, i.e. visible tiling on a large volume and no procedural variation at all on a volume under 10 m; `MinimumQuadTreeSize` (`:35`) is likewise unreachable. Dedup: searched the board for `create_procedural`, `ProceduralFoliageSpawner`, `TileSize`, `minScale`, `alignToNormal` and every `B-foliage-*` / `E-foliage-*` file. `B-foliage-create-procedural-empty-callback-noop` (IN-REVIEW, High) is the same verb but a different defect and is confirmed fixed at HEAD in this tree — the `ProceduralFoliageEditorLibrary` reflection path is present at `:1176-1180` and `instances_spawned` is a measured delta at `:1202-1203`, reported at `:1225` — so this is a follow-on, not a dedup-append to it. `E-foliage-nested-input-schemas-undocumented` (IN-REVIEW) documents the payload shape, not which fields are read, and must be updated in the same commit as any fix here. Nothing on the board mentions the hardcoded tiling.
- `#2-honoured-scale-align-and-tiling` `IN-REVIEW` developer — Honoured, not refused: the mesh build can take all five values, and `foliage.add_type` in the same file already proved it. Every claim in the ticket verified in source first. **Per-type fields** — `foliage.add_type`'s read + validation and its write block are promoted to two file-scope helpers in `FoliageHandler.cpp` (`ReadFoliageScaleAndAlign` / `ApplyFoliageScaleAndAlign`, defined after `GetOrCreateFoliageActorForWorldSafe`); `add_type` now calls them instead of its inline copies, and `create_procedural`'s per-entry loop calls them too, so the two verbs cannot drift. An entry whose range is non-positive or inverted is routed through the existing `NoteSkippedType` channel with the reason rather than failing the batch, and does not consume a `_FT_<n>` index. `minScale`/`maxScale`/`alignToNormal` were removed from `UnreadPerTypeFields` in the same edit; the array survives carrying `randomYaw`, which this verb genuinely still does not read, so `ignoredFields` stays live and honest. **Tiling** — `tileSize` and `numUniqueTiles` are now `RPC_PARAM_OPT` (defaults 1000 cm / 10, unchanged), validated (`INVALID_ARGUMENT` on non-positive / below 1), and the effective pair is always echoed as `tile_size` / `num_unique_tiles` whether supplied or defaulted. The default `TileSize` is deliberately left a constant rather than derived from `bounds`: the ticket lists that as optional, deriving it would change existing callers' scatter silently, and the echo plus the doc now make the grid visible so a caller can raise it themselves. `MinimumQuadTreeSize` is still unreachable and was left alone — out of the ticket's Fix list, and no caller-visible symptom is attributed to it. **Not a declared-but-never-read case**, and the guard's blind spot here is a third one worth naming: the three fields are nested keys inside `foliageTypes[]`, and `Tests/Infra/TestDeclaredParamCoverage.cpp` excludes nested-object reads *by design* (its KNOWN LIMITS section says so), so neither reads-without-declaration nor declarations-without-reads could ever have seen them. `tileSize`/`numUniqueTiles` were absent from both sides, which the dispatcher's unknown-param gate already surfaced honestly as `UNKNOWN_PARAMS`. A guard for accepted-and-inert *nested* keys would need a different mechanism than the source scanner has; worth its own ticket, not filed here. **Files changed:** `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, `Docs/wiki-src/foliage.md` (create_procedural H3 rewritten: which per-entry keys are read, the skip-on-bad-range rule, the tiling pair and its echo, worked example updated), `Source/PinWright/Private/Tests/Infra/TestFoliageNestedInputSchemaDocs.cpp` (the assertion asserting the *old* "not honored" doc claim would have kept passing against text that is now false — replaced with overlay-exclusive markers for the new contract, per `E-foliage-nested-input-schemas-undocumented`'s same-commit requirement). **Regression test:** `Source/PinWright/Private/Tests/World/TestFoliageCreateProceduralTypeConfigHonesty.cpp`, `PinWright.foliage.create_procedural.AppliesScaleAlignAndTiling` — behavioral, routed through `FRpcDispatcher::ProcessRequest` (not `InvokeHandler`, which never runs the unknown-param gate half of the fix). It supplies minScale 0.4 / maxScale 2.5 / alignToNormal:false / tileSize 2500 / numUniqueTiles 3, then **opens the two assets the handler built and reads the properties back** — `ScaleX/Y/Z` and `AlignToNormal` off the generated `UFoliageType_InstancedStaticMesh`, `TileSize`/`NumUniqueTiles` off the `UProceduralFoliageSpawner` — because the response alone could not distinguish the defect. Pre-fix it fails twice over: the dispatcher answers `UNKNOWN_PARAMS`, and with the tiling keys removed the scale readbacks would still read the CDO's 1.0/true. A second `foliageTypes[]` entry with an inverted range asserts the per-entry skip, its reason, and that it generated no `_FT_1`. Not compiled and not run — per instruction, the wave owner builds and runs the suite.
- `#3-scale-write-lands-on-a-property-procedural-never-reads` `OPEN` tester — **Returned on the scale half only. The tiling half and `alignToNormal` are verified fixed and stay fixed.** Measured against a running editor on a four-species vegetation test level; this is the first RPC verification this ticket has had. **Verified working from `#2`:** `tileSize` / `numUniqueTiles` are accepted, validated, applied and echoed (`FoliageHandler.cpp:1340-1347` read + validate, `:1371-1372` write, `:1586-1587` echo as `tile_size` / `num_unique_tiles`); `alignToNormal` is honoured end to end - types built with it tilt to 37.7 deg on slope, types built without it stay vertical, so `#1`'s open question about the procedural rotation path (`InstancedFoliage.cpp:5540-5543`) is answered in the fix's favour. **Returned:** `ApplyFoliageScaleAndAlign` (`FoliageHandler.cpp:117-129`) writes `ScaleX/Y/Z.Min/Max` and `AlignToNormal`, and is called from `create_procedural`'s per-entry loop at `:1449` - but **the procedural placement path never reads `ScaleX/Y/Z`**. `FPotentialInstance::PlaceInstance` reaches `Settings->GetRandomScale()` only inside `if (DesiredInstance.PlacementMode != EFoliagePlacementMode::Procedural)` (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:5508`, the guard; `:5510`, the call). The procedural branch takes `FVector3f(Settings->GetScaleForAge(DesiredInstance.Age))` (`:5516`), and `GetScaleForAge` (`InstancedFoliage.cpp:987`, declared `Runtime/Foliage/Public/FoliageType.h:427`) interpolates `UFoliageType::ProceduralScale` (`FoliageType.h:503`). `ProceduralScale` appears **nowhere** in `FoliageHandler.cpp`, so it stays at its CDO `(1, 3)`. **Repro:** `foliage.create_procedural` with `minScale 0.25` / `maxScale 0.35` -> call succeeds; the generated `_FT_<n>` asset reads back `scale_x (0.25, 0.35)` **and** `procedural_scale (1.0, 3.0)`; the 43 instances actually placed came out at exactly four discrete sizes, `{1.0, 1.2, 1.4, 1.6}` - the age ladder, not the requested range. **`#2`'s regression test cannot see this and is green:** `PinWright.foliage.create_procedural.AppliesScaleAlignAndTiling` asserts `ScaleX/Y/Z` off the generated `UFoliageType_InstancedStaticMesh`, i.e. it asserts exactly the write the simulation ignores. Reading the asset back was the right instinct - `#2` says so explicitly, 'because the response alone could not distinguish the defect' - but the property it reads is not the one that decides. **Severity raised Medium -> High**, argued against the rubric rather than asserted, and using `#1`'s own test: `#1` earned Medium because `ignoredFields` kept the caller honest - 'the caller is neither deceived nor stuck ... nothing they were told is false'. `#2` correctly removed `minScale`/`maxScale` from `UnreadPerTypeFields` in the same edit, which was the right thing to do for the write it believed it had landed; the effect on this defect is that both of `#1`'s Medium conditions have now flipped. The caller is told something false twice - the response no longer lists the keys as ignored, and a `property.get` on the asset returns the range they asked for - and the workaround `#1` named (author the types with `foliage.add_type` first) does not help either, because `add_type` writes the same `ScaleX/Y/Z` through the same shared helper. That is the README's High impact class verbatim: silent wrong data on a normal path, where the caller trusts a result that is a lie and builds on it. **Reach modifier declined in both directions**: `foliage.create_procedural` is one of six verbs in its namespace - not an every-session method, and not a rare edge path - so High stands unmodified. Not Critical: nothing is corrupted or lost, the scatter is merely uniform. **Fix.** Write `ProceduralScale.Min/Max` in `ApplyFoliageScaleAndAlign` alongside `ScaleX/Y/Z` rather than instead of them - both verbs share that helper and `foliage.add_type`'s painting path genuinely needs `ScaleX/Y/Z`, so the helper should set both and the two callers stay identical. One knob more is needed for the fix to be observable at all: `MaxInitialAge` defaults to 0, and at age 0 `GetScaleForAge` returns the range minimum for every instance, so a correct `ProceduralScale` still yields one size until `MaxInitialAge > 0` - that is the `{1.0, 1.2, 1.4, 1.6}` ladder above, and it is why the fix must be verified on placed instances, not on the asset. Make the regression test read `procedural_scale` off the asset **and** assert the spread of the placed instances' `DrawScale3D`; the asset-property assertion alone is what let this through. **Cross-links filed this session.** `B-spawned-volumes-have-no-brush-geometry` - same verb: the `AProceduralFoliageVolume` it spawns has bounds extent `(0,0,0)`, which is why four `create_procedural` calls reported `instances_spawned: 0`; that is also the real reason `B-foliage-create-procedural-empty-callback-noop`'s `#2` saw a zero count, not the 'headless world has no surface' it recorded. `B-create-procedural-density-writes-paint-density` - same verb, same shape: `density` writes `UFoliageType::Density` while the simulation reads `InitialSeedDensity`. `F-procedural-foliage-simulation-knobs-unreachable` - `ProceduralScale` is one of fourteen simulation properties with no path through this verb, and if that ticket is taken first this one closes with it. Also relevant to whoever picks this up: `F-foliage-namespace-has-no-behavioural-tests` leaves `create_procedural` untested *because* this ticket rewrites those lines, so the coverage does not exist to catch the next inert write either.
