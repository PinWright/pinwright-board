---
id: B-asset-dump-scs-omits-inherited-parent
title: "asset.dump scs.json omits inherited parent-class SCS nodes, leaving child component refs dangling"
status: DONE
severity: High
category: bug
tags: [asset-dump, scs, blueprint, components]
---

# `scs.json` drops parent-class SCS nodes, leaving child references dangling

When a Blueprint adds SCS components parented to a root that lives in
a parent Blueprint's SCS (commonly `DefaultSceneRoot` from the parent),
`asset.dump`'s `scs.json` writer iterates only the *current* BP's
own `USimpleConstructionScript::GetAllNodes()` and never includes the
parent-class SCS nodes. Result: child components carry
`"parent": "DefaultSceneRoot"` references that point at a name absent
from the same file's `components` array.

104 of 874 BPs with SCS dumps (11.9%) hit this. 101 of those have
the entire SCS tree dangling — i.e. every component in the file
references a parent that isn't in the file.

This is the likely root manifestation of the existing
`B-component-diff-vs-class-cdo` issue: a consumer comparing
`scs.json` against `properties.json::BlueprintCreatedComponents` or
the CDO sees a mismatch whenever the parent class contributes the
root.

**Repro:**
- `App\App\LevelBlueprints\DynamicObstacles\B_SM_Floor3x3\scs.json` —
  `SM_Cube_Plastic → DefaultSceneRoot` (inherited from
  `B_DroneObstacle`)
- `App\PathTracer\Blueprints\Path_Marks\BP_RaceArrows\scs.json` —
  `TRS → Billboard` (inherited from `BP_PathMark_Master`)

**Workaround:** Cross-reference `meta.json::parentClass` and
recursively load that parent's `scs.json` to assemble the full tree.
Possible in principle but every consumer has to reimplement it.

**Fix (proposed):** Either:
1. Include parent-class SCS nodes in `scs.json`, tagged with
   `"source": "inherited"` so consumers can filter, OR
2. Add an `inheritedRoots` array recording the inherited parent
   names so consumers can stitch the tree without re-reading files.
   Option (1) is more useful; option (2) is cheaper.

Coordinate with `B-component-diff-vs-class-cdo` since both touch the
SCS-vs-CDO comparison code path.

## History
- `#1-initial-audit` `OPEN` reporter — 104 of 874 BPs with `scs.json` (11.9%) have child components referencing parents not present in the same file. Repro on `B_SM_Floor3x3` (parent `B_DroneObstacle`) and `BP_RaceArrows` (parent `BP_PathMark_Master`). Likely root cause behind the existing `B-component-diff-vs-class-cdo` symptom.
- `#2-emit-inherited-scs-nodes` `IN-REVIEW` developer — Added `KSourceInherited = "inherited-scs"` taxonomy and a parent-class chain walk in EditorAutomationRpcGateway_SCSHandlers.cpp::FSCSHandlers::GetBlueprintSCS that emits each ancestor BP's SCS nodes (skipping names already produced by closer sources) with `source` and `inheritedFrom` fields. Test fixture `B_SM_Floor3x3` (parent `B_DroneObstacle`) in TestAssetDumpInheritance.cpp; existing dump caches will need a re-sweep via `asset.dump_folder` to refresh the 104 affected files.
- `#3-verify-scs-inheritance` `DONE` tester — Re-dumped `B_SM_Floor3x3` via `asset.dump`; `scs.json` now contains the inherited `DefaultSceneRoot` node with `source: "inherited-scs"` and `inheritedFrom: "/App/App/LevelBlueprints/Obstacles/B_DroneObstacle.B_DroneObstacle"`. The `SM_Cube_Plastic → DefaultSceneRoot` parent ref no longer dangles. Fix confirmed.
