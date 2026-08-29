---
id: B-create-procedural-ignores-scale-and-normal-fields
title: "foliage.create_procedural builds each UFoliageType with only mesh and density — it drops per-type minScale/maxScale/alignToNormal (honestly, into ignoredFields) and hardcodes the spawner's TileSize and NumUniqueTiles (silently, with no echo at all), so every plant is the same size, upright, and tiled on a 10 m grid"
status: OPEN
severity: Medium
category: bug
tags: [foliage, create_procedural, procedural-foliage-spawner, scale, align-to-normal, tile-size, hardcoded, ignored-fields, response-honesty, vegetation]
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
