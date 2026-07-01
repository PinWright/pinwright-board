---
id: E-open-level-blueprint-unusable-assetpath
title: "level.structure.open_level_blueprint returns assetPath=/Game/Maps/<Map> (bare package path) that every blueprint.graph.* method rejects with ASSET_NOT_FOUND — the usable level-script object path is never surfaced"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level-structure, level-blueprint, blueprint-graph, assetpath, misleading-result, round-trip]
---

# `open_level_blueprint` hands back an `assetPath` that the blueprint.graph.* methods it is the prerequisite for cannot consume

`level.structure.open_level_blueprint` succeeds and returns
`assetPath: "/Game/Maps/ExampleProjectWelcome"` (the bare **package** path) as
the level-script Blueprint's handle. Its own wiki page frames the verb as the
"Required prerequisite for some level-blueprint node-graph operations" — i.e. the
caller is expected to open the level BP, then author its graph with the
`blueprint.graph.*` family. But the `assetPath` value the verb reports is
**non-functional** for that family: every `blueprint.graph.*` method rejects the
bare package path with a hard `[ASSET_NOT_FOUND]`, because a level-script
Blueprint cannot be `LoadObject`'d at a `.umap` package path — it lives at the
sub-object path `…:PersistentLevel.<MapName>`.

The only value that works with `blueprint.graph.*` is the full object path
`/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.ExampleProjectWelcome`,
which `open_level_blueprint` **never surfaces** — the caller has to know the
level-script sub-object path shape on their own (the attempt agent recovered it
only by reading the plugin C++). To compound the trap, both `list_graphs` and
`create_node`, when handed the *working* full object path, echo
`assetPath: "/Game/Maps/ExampleProjectWelcome"` (the bare form) back in their
success bodies — so the success responses also advertise the path that fails.

The individual calls each behave correctly in isolation (the seed really did
open the level BP; `ASSET_NOT_FOUND` is an accurate description of "no Blueprint
loadable at that package path"). The defect is the **misleading handoff**: a
reasonable open-then-author flow takes the `assetPath` the seed reports, feeds it
to the next documented step, and hits a hard error with zero hint that a
different path form is required. This is the "result misreports a field /
concretely misleading" ergonomic class, not a crash or a wrong result.

## What it should do

Either (preferred) `level.structure.open_level_blueprint` should return the
**usable** level-script object path (the `…:PersistentLevel.<MapName>` form) as
`assetPath` (or as an explicit additional field such as `blueprintObjectPath`),
so the value it hands back round-trips into `blueprint.graph.*`; and/or the
blueprint.graph.* path resolver should accept a `.umap` package path that
contains exactly one `LevelScriptBlueprint` and resolve it to that sub-object,
the way `editor.open_level` / `level.structure.get_level_structure_info` already
accept the bare map path. At minimum the seed must not advertise a handle its
own documented downstream operations reject.

## Verbatim repro (live, replay-confirmed, ExampleProjectWelcome)

```
call("editor.open_level", {"levelPath":"/Game/Maps/ExampleProjectWelcome"})
-> ok {"alreadyLoaded":true,"levelPath":"/Game/Maps/ExampleProjectWelcome", ...}

call("level.structure.open_level_blueprint", {})
-> ok {"assetPath":"/Game/Maps/ExampleProjectWelcome",          # <- bare package path handed back
       "assetName":"ExampleProjectWelcome",
       "assetClass":"LevelScriptBlueprint","existsAfter":true,"levelName":"ExampleProjectWelcome"}

# Reuse the assetPath the seed just reported:
call("blueprint.graph.create_node",
     {"assetPath":"/Game/Maps/ExampleProjectWelcome","nodeType":"Event",
      "eventName":"ReceiveBeginPlay","x":200,"y":200})
-> [ASSET_NOT_FOUND] Could not load blueprint at path: /Game/Maps/ExampleProjectWelcome

call("blueprint.graph.list_graphs", {"assetPath":"/Game/Maps/ExampleProjectWelcome"})
-> [ASSET_NOT_FOUND] Could not load blueprint at path: /Game/Maps/ExampleProjectWelcome

# Only the full level-script object path works — and it is never surfaced by the seed:
call("blueprint.graph.list_graphs",
     {"assetPath":"/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.ExampleProjectWelcome"})
-> ok {"graphs":[{"name":"EventGraph","kind":"ubergraph",...}],
       "assetPath":"/Game/Maps/ExampleProjectWelcome", ...}   # success body STILL echoes the bare form
```

## Impact

A natural "open the level BP, then wire its event graph" task (BeginPlay ->
Print String, etc.) stalls on the first authoring call. The blocker is fully
recoverable — the agent eventually guessed/derived the `…:PersistentLevel.<Map>`
object path — but only after reading the handler C++; nothing in the seed result
or the wiki hints at it, so a fresh caller burns at least one `ASSET_NOT_FOUND`
round-trip and has to reverse-engineer the level-script path shape.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed on live editor (ExampleProjectWelcome). `level.structure.open_level_blueprint` returns `assetPath:"/Game/Maps/ExampleProjectWelcome"` (bare package path); reusing that value in `blueprint.graph.create_node` and `blueprint.graph.list_graphs` both hard-fail `[ASSET_NOT_FOUND] Could not load blueprint at path: /Game/Maps/ExampleProjectWelcome`. The only form that resolves is the full level-script object path `/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.ExampleProjectWelcome`, which the seed never surfaces; both list_graphs and create_node still echo the bare `assetPath` in their success bodies even when given the working object path. Distinct from the `path` vs `assetPath` param-NAME drift cluster (`E-blueprint-param-name-path-vs-assetpath` DONE, `E-asset-path-vs-assetpath-list-drift`): the param name `assetPath` is correct here — this is a returned VALUE that does not round-trip into the documented downstream operations, for level-script Blueprints specifically.
- `#2-fix` `IN-REVIEW` developer — Root-caused to the shared `AddAssetVerification` helper deriving `assetPath` from `Asset->GetPackage()->GetPathName()` — fine for normal top-level assets (the package path IS the loadable object path) but wrong for a `ULevelScriptBlueprint`, which is a sub-object of the `.umap` so its package path is the bare map path that `LoadObject<UBlueprint>` cannot resolve. Fix special-cases `ULevelScriptBlueprint` in `AddAssetVerification` (`Source/PinWright/Private/Utils/AssetUtils.cpp`, via new `ResolveVerificationAssetPath`) to emit the full object path `…:PersistentLevel.<Map>` (`Asset->GetPathName()`), which round-trips — this fixes BOTH the seed's `assetPath` AND every `blueprint.graph.*` success body (`list_graphs`/`create_node`/…) at once, with zero blast radius for non-level-script assets. `level.structure.open_level_blueprint` (`Source/PinWright/Private/Handlers/Level/LevelStructureHandler.cpp`) additionally surfaces the usable handle explicitly under `blueprintObjectPath` and the bare `.umap` package path under `mapPath`, mirroring the actorPath/mapPath split of the sibling `E-actor-verification-actorpath-is-map-path`. Wiki overlay (`Docs/wiki-src/level.structure.md`) gains an `### level.structure.open_level_blueprint` section documenting the round-trippable `assetPath`/`blueprintObjectPath` vs the non-loadable `mapPath`. Regression test `PinWright.level.structure.open_level_blueprint.AssetPathRoundTrips` (`Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp`) drives the production `AddAssetVerification` against the live editor world's level-script Blueprint and asserts the emitted `assetPath` is NOT the bare package path and `LoadObject<UBlueprint>`-resolves back to the same Blueprint (the exact downstream round-trip); reverting the helper to `GetPackage()->GetPathName()` returns null and fails it.
