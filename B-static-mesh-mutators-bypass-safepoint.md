---
id: B-static-mesh-mutators-bypass-safepoint
title: "asset.nanite_rebuild_mesh and static_mesh.bake_transform run full StaticMesh rebuilds without a safe-point gate"
status: OPEN
severity: High
category: bug
tags: [static-mesh, nanite, bake-transform, safe-point, tick, compile, crash]
---

# Two StaticMesh rebuild verbs can execute inside an engine tick

## What's wrong

`asset.nanite_rebuild_mesh` changes Nanite settings, invokes the rebuild path, and drains static
mesh compilation at `AssetWorkflowHandler.cpp:1364-1392`. `static_mesh.bake_transform` calls
`PostEditChange` and `FinishCompilation` at `StaticMeshBakeTransformHandler.cpp:252-256` after
rewriting mesh descriptions, collision, and sockets. Neither verb is in
`Dispatch/SafePoint.cpp` and neither calls `RunAtSafePoint`.

These paths release/recreate render resources, enqueue and wait for compilation, and can flush
render work. If an RPC is drained during `UWorld::Tick`, the entire rebuild lands on that tick
stack. This is the same High crash mechanism already documented for StaticMesh rebuild verbs;
there is no direct kill reproduced through these two names, so severity is discounted from the
fatal impact class.

## What it should do

Add dispatcher safe-point entries for both verbs (neither has a cross-dispatch caller), and add a
source ratchet requiring every in-place StaticMesh build/`PostEditChange`/compilation drain to be
safe-pointed.

## Workaround

There is no reliable caller-side way to know whether the dispatcher is currently inside a tick.

## Related

- `B-nested-gamethread-marshal-defeats-tick-gate` fixed `asset.generate_lods` but not these verbs.
- `B-set-lod-settings-unguarded-rebuild` is the Geometry-module sibling.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the tick-unsafe catalog; no editor, build, test, or RPC run was performed.
