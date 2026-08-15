---
id: B-nav-agent-properties-restored-on-registration
title: "navigation.set_nav_agent_properties writes ARecastNavMesh::AgentRadius/AgentHeight, which SetConfig restores from NavDataConfig at every registration"
status: OPEN
severity: Medium
category: bug
tags: [navigation, navmesh, derived-state, persistence, silent-revert, misleading-success]
---

# `navigation.set_nav_agent_properties` writes a value the engine re-derives on load

Same shape as `B-spline-point-scale-water-derived`: the write lands, reads back, saves, and is
recomputed away — invisible to every check short of a genuine reload.

`Handlers/AI/NavigationHandler.cpp:277-278` assigns `ARecastNavMesh::AgentRadius` and
`AgentHeight` directly on the level's default nav data (verb registered at `:241`), calls
`MarkPackageDirty()` at `:288`, and echoes the mutated fields back at `:289-290`.
`navigation.get_navigation_info` re-reads the same fields at `:872-873`, so a set-then-verify
loop always passes.

## Root cause (source-verified against `C:\UE_5.8`)

`ARecastNavMesh::SetConfig(const FNavDataConfig&)`
(`Runtime/NavigationSystem/Private/NavMesh/RecastNavMesh.cpp:1405`) assigns
`AgentHeight = NavDataConfig.AgentHeight; AgentRadius = NavDataConfig.AgentRadius`
(`:1420-1421`). `UNavigationSystemV1::RegisterNavData` (`NavigationSystem.cpp:2843`) calls it
on every registration (`:2920`, single-agent path `:2875`), and registration runs when a level
is added to the world (`:5016`) and at nav-system init (`:1412`, `:1658`).

The handler never writes `NavDataConfig`, and `RegisterNavData` reads the config through
`NavData->GetConfig()` (`NavigationData.h:657`), so `IsEquivalent` still matches, the navmesh
is kept rather than discarded, and the project value is quietly restored.

The engine says so out loud at `RecastNavMesh.cpp:4238-4246`: *"Changing AgentRadius directly
on RecastNavMesh instance is unsupported. Please use Project Settings > NavigationSystem >
SupportedAgents to change AgentRadius"* — a `PostEditChangeProperty` warning the handler never
triggers, because it assigns the member rather than raising a property-change event.

## Authoritative state

`UNavigationSystemV1::SupportedAgents[i].AgentRadius` / `.AgentHeight` (Project Settings →
Navigation System → Supported Agents), then `ANavigationData::SetConfig` + re-register. There
is no per-instance setter.

## Scoped out (verified, so the fix does not over-reach)

Sibling writes in the same handler are **not** this defect: `AgentMaxSlope`
(`NavigationHandler.cpp:279`) is untouched by `SetConfig`; `agentStepHeight` →
`NavMeshResolutionParams[Default].AgentMaxStepHeight` (`:282`) is only overwritten when
`Src.HasStepHeightOverride()` (`RecastNavMesh.cpp:1424-1431`); the
`navigation.configure_nav_mesh_settings` writes at `:185-224` (`TileSizeUU`, `MinRegionArea`,
`MergeRegionSize`, `MaxSimplificationError`, `CellSize`, `CellHeight`) have no load-time
re-derive found.

## Fix shape

Follow `Docs/rpc-design.md` §5a: write the project `SupportedAgents` entry and re-register, or
refuse with `DERIVED_PROPERTY` plus a `derivedWrite` payload naming the authoritative route.
Do not silently keep writing the instance field.

## History
- `#1-found-by-sweep` `OPEN` reporter — Found by the derived-state sweep run alongside `B-spline-point-scale-water-derived`. Not reproduced live; the mechanism is read out of UE 5.8 source at the lines quoted above. Confidence high: the engine ships an explicit "changing AgentRadius directly on RecastNavMesh instance is unsupported" warning for exactly this write.
