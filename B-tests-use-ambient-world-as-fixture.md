---
id: B-tests-use-ambient-world-as-fixture
title: "Four tests use the ambient editor world as their fixture (its lights, its OFPA flag, its World Partition), so on the suite's blank world they skip and measure nothing"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, testing, fixture-skip, blank-world, environment, volume, world-partition]
encounters: 1
lastSeen: 2026-09-29T17:16:39Z
---

# Tests take their fixture from whatever world is open

`PinWright.aa_suite_start.OpenBlankTransientWorld` swaps the host's startup map for a blank untitled
world. Four tests never built their own fixture and relied on a property of the map that used to be
open, so all four skip with `PINWRIGHT_ASSERTIONS_SKIPPED` (suite log
`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log`):

- `PinWright.environment.control.set_skylight_intensity.AppliesOnStaticMobility` (`:29013`, "No ASkyLight in the editor world") and
  `PinWright.environment.control.set_sun_intensity.AppliesOnMovableMobility` (`:29027`, "No ADirectionalLight in the editor world"),
  `Tests/World/TestEnvironmentHandlers.cpp`: iterated the world for the level's own light; the header said a spawned fixture "would not work" because the handler picks the first light.
- `PinWright.volume.create_nav_mesh_bounds_volume.MarksLevelPackageDirty` (`:52886`, `actor-shares-level-package`), `Tests/World/TestVolumeHandlers.cpp`: needs One-File-Per-Actor so the spawned volume has its own package; the blank level does not use external actors.
- `PinWright.world_partition.create_datalayer.EchoesNonTransientPath` (`:54700`, `no-partitioned-world`), `Tests/World/TestCreateDatalayerPersistable.cpp`: the handler returns `NOT_PARTITIONED` before building a package on a non-partitioned world.

The rest of `environment.*` was checked: the `*.ValidParamsNoCrash` tests assert only handler registration and do not skip; every other skip site in `Tests/Environment/` and `TestEnvironmentHandlers.cpp` is `no-editor-world`, a cvar gate, or spawns its own fixture.

## History
- `#1-ambient-world-fixture` `OPEN` reporter — Four tests skip on the suite's blank world because they read lights / OFPA / World Partition off the ambient editor world instead of building a fixture.
- `#2-tests-build-own-world-fixture` `IN-REVIEW` developer — `TestEnvironmentHandlers.cpp`: new `FindOrSpawnFirstLight<TLight>` returns the first valid light in `TActorIterator` order (the handlers' resolution) and spawns a GUID-labelled normal level light (`SpawnActorInActiveWorld`, not `RF_Transient`) when none exists; both tests declare `FScopedEditorWorldActorGuard` and fail instead of skipping when no light can be resolved or spawned; header comment corrected. `TestVolumeHandlers.cpp`: `MarksLevelPackageDirty` calls `PersistentLevel->SetUseExternalActors(true)` for the spawn (so `UWorld::SpawnActor` creates an external actor package, `LevelActor.cpp:555-617`) and restores the flag on scope exit after the volume is destroyed; the `actor-shares-level-package` skip stays for a host where externalization still does not happen. `TestCreateDatalayerPersistable.cpp`: `EchoesNonTransientPath` opens a World Partition world with `GEditor->NewMap(true)` behind `FScopedEditorWorldMapGuard` and the shared `ProbeResidentWorldSurvivors` gate (`map-swap-refused` skip only when that gate refuses); any handler error or an "already exists" answer is now a failure; header COVERAGE LIMIT note updated (the echo still asserts the computed path string, not the asset's outer). `-SingleFile` compile of all three files succeeded; not run.
