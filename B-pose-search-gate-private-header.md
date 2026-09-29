---
id: B-pose-search-gate-private-header
title: "pose_search.* reports PLUGIN_DISABLED on UE 5.3-5.5 because the compile gate requires PoseSearchFeatureChannel_Position.h, a Private header before 5.6"
status: OPEN
severity: Medium
category: bug
tags: [pose-search, motion-matching, engine-version, compile-gate, plugin-gated, ue53, ue54, ue55, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# pose_search.* is compiled out on UE 5.3-5.5

PinWright claims 5.3 to 5.8 support, and the `pose_search` wiki page says the namespace needs only
the PoseSearch plugin enabled. On 5.3, 5.4 and 5.5 every `pose_search.*` call fails with
`PLUGIN_DISABLED` "PoseSearch headers are not available in this engine build." even with the plugin
enabled.

The gate, `Source/PinWrightPoseSearch/Private/Handlers/PoseSearch/PoseSearchHandler.cpp:18-25`
(plugin HEAD `71c91649`):

```cpp
#if __has_include("PoseSearch/PoseSearchSchema.h") && __has_include("PoseSearch/PoseSearchDatabase.h") && __has_include("PoseSearch/PoseSearchFeatureChannel_Position.h")
```

Where the channel header lives per engine:

| Engine | Plugin root | `PoseSearchFeatureChannel_Position.h` |
|---|---|---|
| 5.3 | `Engine/Plugins/Experimental/Animation/PoseSearch` | `Source/Runtime/Private/` |
| 5.4, 5.5 | `Engine/Plugins/Animation/PoseSearch` | `Source/Runtime/Private/` |
| 5.6, 5.7, 5.8 | `Engine/Plugins/Animation/PoseSearch` | `Source/Runtime/Public/PoseSearch/` |

On 5.5 the Public `PoseSearch/` folder has no `PoseSearchFeatureChannel_*.h` at all, only the base
`PoseSearchFeatureChannel.h`. A Private header of another module is not on the include path, so
`__has_include` is false, `MCP_HAS_POSESEARCH` is 0 and `EnsurePoseSearchAvailable` (`:101-116`)
takes the `#else` branch for all three verbs. The Build.cs does find the plugin on every version
(`Source/PinWrightPoseSearch/PinWrightPoseSearch.Build.cs:28-32` lists both roots), so the module
builds and registers; only the handler bodies are empty. The class itself is exported on the old
engines (`class POSESEARCH_API UPoseSearchFeatureChannel_Position` in the Private header on 5.3 and
5.5), so it is reachable through reflection.

The error text also misleads: "headers are not available" reads as a broken install, and
`PLUGIN_DISABLED` tells the caller to enable a plugin that is already enabled.
`docs/engine-version-support.md` has no Pose Search row.

Engines: 5.3, 5.4, 5.5 affected; 5.6 to 5.8 fine.

**Fix:** gate on `PoseSearchSchema.h` and `PoseSearchDatabase.h` only. Create the Position channel
via `FindObject<UClass>`/`StaticLoadClass` on `/Script/PoseSearch.PoseSearchFeatureChannel_Position`
plus `NewObject(Schema, Class)`, and set `Bone`, `OriginBone`, `SampleTimeOffset`,
`OriginTimeOffset` and `Weight` through `FProperty` lookups, keeping the typed path under a
`>= 5.6` guard if preferred. If some engine truly cannot support a verb, return a distinct code
(for example `ENGINE_VERSION_UNSUPPORTED`) naming the version, and add a row to
`docs/engine-version-support.md`. Cover with the existing
`FPoseSearchSchemaDatabaseAuthoringPipelineTest` in the version matrix.

**Related:** `F-pose-search-database-authoring` (DONE, added the namespace),
`F-pose-search-schema-channel-kinds` (only Position channels are accepted),
`B-pose-search-create-save-no-disk-write`.

## History
- `#1-private-header-gate` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis and re-verified at plugin HEAD `71c91649` against local 5.3 to 5.8 engine trees: the `__has_include` gate at `PoseSearchHandler.cpp:18` requires `PoseSearch/PoseSearchFeatureChannel_Position.h`, which is under `Source/Runtime/Private/` on 5.3 (Experimental root), 5.4 and 5.5 and Public only from 5.6. Not run live on 5.3-5.5. Severity Medium: a hard blocker for the whole namespace on half the supported engines (High-or-Medium class), bumped down for reach since Pose Search authoring is a rare path.
