---
id: B-level-create-makes-wp-map
title: "level.create is documented 'non-World-Partition' but calls NewMap(true), producing a World Partition map with no persistent .umap"
status: IN-REVIEW
severity: High
category: bug
tags: [level, create, world-partition, newmap, docs-contradicts-behavior, add-to-world]
---

# level.create claims non-World-Partition but builds a World Partition map

`level.create` is both documented (registration description + generated wiki
`level.create.md`) and named to produce a **non-World-Partition** level:

> "Create a new empty (non-World-Partition) level asset. For World-Partition
> levels use level.structure.create_level instead."
> (`LevelHandler.cpp:344`)

But the handler body builds a **World-Partition** world. At
`LevelHandler.cpp:402` it calls:

```cpp
if (UWorld* NewWorld = GEditor->NewMap(true))
```

The engine signature is `UWorld* NewMap(bool bIsPartitionedWorld = false)`
(`Editor/UnrealEd/Classes/Editor/EditorEngine.h:2007`). Passing `true` means
`bIsPartitionedWorld = true`, so `level.create` does exactly the opposite of
what the verb that it tells you to use for WP (`level.structure.create_level`)
is for. The documented input wrongly produces a WP map.

## Concrete harm (the cascade)

A WP map is not a single self-contained persistent `.umap`: actors live in
`__ExternalActors__/...` and the map carries HLOD side-assets. The result of the
documented "non-WP" call is therefore unusable as a classic streaming sublevel,
and it never lands a persistent `.umap` on disk:

- The created world has **13 actors** (WP scaffold: WorldPartition,
  WorldDataLayers, HLOD setup), not the ~2 of a truly empty non-WP level.
- On disk only `<name>_HLODLayer_Instanced.uasset` / `_Merged.uasset` plus an
  `__ExternalActors__/Maps/<name>/` directory appear — **no `<name>.umap`**,
  even though `level.create` itself runs `FEditorFileUtils::SaveMap` (`:413`)
  and reports success.
- A subsequent `level.save` returns `{saved:true}` but still writes **no
  persistent `.umap`**.
- `level.add_to_world` (which requires the physical streaming `.umap`) then
  fails `[PACKAGE_NOT_FOUND] Level file not found: <path>`, so the documented
  "make a non-WP level to stream as a sublevel" workflow is impossible via the
  verb the docs point you at.

## What it should do

Either (a) honor the documentation and call `GEditor->NewMap(false)` so
`level.create` produces a genuine non-World-Partition level (a single persistent
`.umap` usable as a streaming sublevel — which is what its own description and
the seed task expect), or (b) if a partitioned default is intended, fix the
registration string + generated wiki so they no longer claim "non-World-Partition"
and stop directing WP work elsewhere.

Note this is distinct from `B-create-level-saved-true-no-umap` (IN-REVIEW): that
ticket is about the **`level.structure.create_level`** RPC (`LevelStructureHandler.cpp`)
and its `McpSafeLevelSave` OR-policy falsely reporting `saved:true`. This ticket
is about the **`level.create`** RPC (`LevelHandler.cpp:402`) building a WP world
at all, in direct contradiction of its own "non-World-Partition" contract — a
different handler and a different root cause (`NewMap(true)`).

## Repro (replayed via mcp__editor-automation__call)

1. `level.create {levelPath:"/Game/Maps/OracleReproAnnex"}`
   → success `{levelPath, packagePath, objectPath}`.
2. `level.get_info {}` (active world is now the new map) → `actorCount: 13`
   (WP scaffold; a true empty non-WP level would have ~2).
3. On-disk check of `Content/Maps/`: only
   `OracleReproAnnex_HLODLayer_Instanced.uasset` + `OracleReproAnnex_HLODLayer_Merged.uasset`
   and an `__ExternalActors__/Maps/OracleReproAnnex/` directory — **no
   `OracleReproAnnex.umap`**.
4. `level.save {}` → job completes `{saved:true}`; on-disk re-check: still **no
   `OracleReproAnnex.umap`** (only the two HLOD side-assets).
5. `level.load {levelPath:"/Game/Maps/ExampleProjectWelcome"}` (back to persistent
   world), then `level.add_to_world {levelPath:"/Game/Maps/OracleReproAnnex"}`
   → `[PACKAGE_NOT_FOUND] Level file not found: /Game/Maps/OracleReproAnnex`.

Root cause: `LevelHandler.cpp:402` `GEditor->NewMap(true)` (`bIsPartitionedWorld=true`)
vs the registration's "non-World-Partition" claim at `LevelHandler.cpp:344`.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `level.create {levelPath:"/Game/Maps/OracleReproAnnex"}` against the live editor: returned success; the new active world had 13 actors (WP scaffold), and `Content/Maps/` got only `OracleReproAnnex_HLODLayer_Instanced.uasset` + `_Merged.uasset` + an `__ExternalActors__/Maps/OracleReproAnnex/` dir, with NO `OracleReproAnnex.umap`. A follow-up `level.save` returned `{saved:true}` but still wrote no `.umap`; `level.add_to_world` on the path then failed `[PACKAGE_NOT_FOUND] Level file not found: /Game/Maps/OracleReproAnnex`. Code-confirmed: `LevelHandler.cpp:344` documents/names the verb "non-World-Partition" while the body at `LevelHandler.cpp:402` calls `GEditor->NewMap(true)` = `bIsPartitionedWorld=true` (engine sig `EditorEngine.h:2007`), producing a WP map. Dedup: distinct from `B-create-level-saved-true-no-umap` (IN-REVIEW — different RPC `level.structure.create_level` in `LevelStructureHandler.cpp`, McpSafeLevelSave OR-policy) and from `E-level-create-name-path-alias` (OPEN — param-name aliasing). No board entry references `NewMap`, "non-World-Partition", or `bIsPartitionedWorld` (ripgrep clean).
- `#2-fix-newmap-false` `IN-REVIEW` developer — Root-cause fix (option a): changed `GEditor->NewMap(true)` → `GEditor->NewMap(false)` in `Handlers/Level/LevelHandler.cpp` (the former `:402`) so `level.create` honors its documented "non-World-Partition" contract and produces a single self-contained, saveable persistent `.umap` usable as a streaming sublevel — the registration string and the `level.structure.create_level` WP escape hatch are now consistent with the verb's behavior. Added an explanatory comment at the call site. Regression test `EditorAutomationRpcGateway.level.create.ProducesNonPartitionedWorld` in `Private/Tests/World/TestLevelHandlers.cpp` invokes the real `level.create` handler against a GUID-unique throwaway `/Game/Maps/__EARG_WPRegressionProbe_*` path (the unique name avoids the "package exists → Open" short-circuit) and asserts the world it leaves active is `!IsPartitionedWorld()`; it fails if the `NewMap` arg is reverted to `true`. The test restores the originally-open map and deletes the throwaway asset, and degrades to registration-only coverage when no editor world/subsystem is available. Not yet compiled/run (later phase). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/World/TestLevelHandlers.cpp`.
