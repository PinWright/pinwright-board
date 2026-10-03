---
id: B-asset-dump-tier3-determinism-backlog
title: "Tier-3 dump-determinism backlog: render-data-derived static-mesh stats, CPF_Transient in DataTable/StateTree rows, auto-suffixed identities in PhysicsAsset/SoundCue"
status: OPEN
severity: Low
rice: [1, 1, 1, 2]
priority: 4
category: bug
tags: [asset-dump, determinism, backlog]
---

# Tier-3 dump-determinism backlog: render-data-derived static-mesh stats, CPF_Transient in DataTable/StateTree rows, auto-suffixed identities

Residual nondeterminism left by the July 2026 dump-determinism audit. None of these flip bytes on a
routine re-dump of an unchanged asset within one engine build. They show up after an engine or DDC
rebuild, from editor-transient state, or when an asset is re-authored or duplicated.

1. **Static-mesh stats come from built render data.** `static_mesh.json` reads `bounds`
   (`GetExtendedBounds`, `Source/PinWright/Private/Handlers/Asset/StaticMeshDumpBuilder.cpp:31`),
   `trianglesByLod` / `verticesByLod` (`:49-55`), and the per-LOD `sections[]`, `uvChannelsByLod`,
   per-section and per-slot `boundingBox` and `slotUsage[]` (`RenderData->LODResources`, `:64-127`).
   A DDC or engine rebuild can change these with no source-asset edit. SkeletalMesh reads the
   imported model and is not affected.
2. **Full-reflection row serialization doesn't skip `CPF_Transient`.** DataTable rows
   (`Source/PinWright/Private/Handlers/DataTable/DataTableDumpBuilder.cpp:19`) and StateTree instanced data
   (`Source/PinWright/Private/Handlers/Asset/StateTreeDumpBuilder.cpp:53`) call `FJsonObjectConverter::UStructToJsonObject`
   with `CheckFlags = 0, SkipFlags = 0`, so transient properties land in the sidecar.
3. **Auto-suffixed instance names are used as identity.** PhysicsAsset `bodies[].name`
   (`PhysicsAssetDumpBuilder.cpp:59`, `BodySetup->GetName()`), SoundCue node `name`
   (`SoundCueDumpBuilder.cpp:34`) and SoundCue `edges[]` (`:48`, child `GetPathName()`) key on
   engine-assigned names like `SkeletalBodySetup_2`, so a semantically identical re-authored or
   duplicated asset gets different identities.

**Fix:** fix opportunistically. Item 1: either drop the render-data fields from the dump (keep them on
`static_mesh.describe`) or document them as build-dependent. Item 2: pass `CPF_Transient` as
`SkipFlags`. Item 3: key on stable data (bone name for bodies; graph position or a stable index for
SoundCue nodes). Every fix changes serialized bytes for its aspect and must bump that aspect in the
`GetAspectVersion` table (`AssetDumpCache.cpp:705-879`) in the same commit.

**Acceptance:** item 2: a DataTable row struct with a `Transient` UPROPERTY dumps without that field.
Item 3: two PhysicsAssets / SoundCues built in a different creation order dump identical `name` /
`edges` values. Item 1: the docs or the dump no longer imply that the render-data fields are
source-stable.

## History
- `#1-filed-from-determinism-audit` `OPEN` reporter - Filed as the consolidated Tier-3 backlog from the dump-determinism audit; all four items carry file:line references from the audit pass. None churn routine same-build re-dumps, hence one Low ticket instead of four.
- `#2-set-ordering-fixed` `OPEN` developer - Partial implementation: generic `FSetProperty` values now sort by canonical serialized JSON before emission, and SoundCue concurrency paths are sorted. The ticket remains OPEN because its other Tier-3 items are unchanged.
- `#3-rephrased` `OPEN` developer — Dropped item 4 (TSet order), fixed in `#2` (`PropertyExport.cpp:1072-1107` sorts set elements). Dropped Cascade from item 2: `CascadeDumpBuilder.cpp:78` now uses `BuildSparsePropertyDiffJson`, which excludes `CPF_Transient` (`Utils/PropertyDiff.h:12`). Item 1 now also lists the render-data-derived `sections[]`, `uvChannelsByLod`, `slotUsage[]` and `boundingBox` fields added since. Refreshed all citations (DataTable is under `Handlers/DataTable/`), named the SoundCue `edges[]` path-name keying (`SoundCueDumpBuilder.cpp:48`), and added per-item Fix and Acceptance. Severity unchanged (Low).
