---
id: B-level-bp-verbs-never-create-level-bp
title: "level.structure.open_level_blueprint / add_level_blueprint_node pass bDontCreate=true, so they fail OPERATION_FAILED on any level whose Level Blueprint was never created"
status: IN-REVIEW
severity: High
category: bug
tags: [level, level-blueprint, handler, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:25:05Z
---

# Level-Blueprint verbs never create the Level Blueprint

`ULevel::GetLevelScriptBlueprint(bool bDontCreate = false)` (`Engine/Classes/Engine/Level.h:1394`).
`LevelStructureHandler.cpp` calls it with `true` in `open_level_blueprint` and
`add_level_blueprint_node`, reading the flag as "create". On a level whose Level Blueprint does not
exist yet (a new map, the untitled world, the suite's blank start world) both verbs return
`OPERATION_FAILED` ("Failed to get Level Blueprint" / "Level Blueprint unavailable for unsaved
levels. Please save the level first.") where the editor itself creates it on open. Reproducible, not
environmental: a host condition only hides it when the startup map already carries a Level Blueprint.

Observed: `PinWright.level.structure.add_level_blueprint_node.HonorsNodeNameAndFlatPos` skipped
`reason=handler-call-failed` (OPERATION_FAILED) and
`PinWright.level.structure.open_level_blueprint.AssetPathRoundTrips` skipped
`reason=level-script-blueprint-unavailable` on the full offscreen suite
(`Saved/Logs/pw_gapwave_full_offscreen2.log:37705`, `:37983`); both run on the blank untitled world
`aa_suite_start` opens. The round-trip test makes the same `GetLevelScriptBlueprint(true)` misread.

**Fix:** pass `false` (create when missing) at both handler call sites and in the round-trip test.

## History
- `#1-bdontcreate-misread` `OPEN` reporter — Filed from the full offscreen suite: both level-BP live tests skipped instead of measuring. Root cause is the inverted `bDontCreate` argument, not the host.
- `#2-pass-false-to-create` `IN-REVIEW` developer — `Handlers/Level/LevelStructureHandler.cpp`: `open_level_blueprint` and `add_level_blueprint_node` now call `GetLevelScriptBlueprint(false)` (creates when missing), with a comment naming the parameter. `Tests/World/TestLevelHandlers.cpp` `open_level_blueprint.AssetPathRoundTrips` fixed the same way. `-SingleFile` compile: clean. Follow-ups not done: `open_level_blueprint`'s "unavailable for unsaved levels, save first" message is now reachable only if creation itself fails and misstates the cause; `add_level_blueprint_node.ValidParamsNoCrash` now really adds an unbound `K2Node_Event` stub to the suite world's Level Blueprint and never removes it (it did the same to the host map's Level Blueprint before the blank start world).
- `#3-test-cleanup-and-honest-message` `IN-REVIEW` developer — Closed both follow-ups in #2. (a) `Tests/World/TestLevelHandlers.cpp`: new `FPWLevelBlueprintNodeGuard` creates-or-fetches the current level's Level Blueprint the way the verbs do, snapshots every graph's nodes and the level package dirty flag, and on scope exit removes each node added since (marking the Blueprint modified so the next PIE compile sees the clean graph) and restores the flag. It is used by `add_level_blueprint_node.ValidParamsNoCrash` and `connect_level_blueprint_nodes.ValidParamsNoCrash`. `HonorsNodeNameAndFlatPos` already removes its probe node and restores the flag; `AssetPathRoundTrips` adds no nodes and restores the flag. (b) `open_level_blueprint`: the dead "unsaved levels, save first" branch and its `bIsSavedLevel` local are removed. The one remaining failure says the level has no Level Blueprint and the engine did not create one, naming the level package. `docs/wiki-src/level.structure.md` states that both verbs create the Level Blueprint when missing. `-SingleFile` compile: clean.
