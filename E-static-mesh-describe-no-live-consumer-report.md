---
id: E-static-mesh-describe-no-live-consumer-report
title: "Nothing reports which live components reference a static mesh, so a caller cannot see in advance that a rebuild will be refused"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [static_mesh, describe, model-compile, niagara, live-consumer, discoverability, crash-adjacent]
encounters: 1
lastSeen: 2026-08-27
---

# The rebuild guard knows; the read does not

`model.compile` now refuses with `MESH_REBUILD_CONSUMER_NOT_QUIESCABLE` when a live component whose
scene proxy caches the mesh's render data cannot be quiesced
(`B-model-compile-live-niagara-mesh-renderer-raytracing-assert`).

The refusal is the first time a caller learns such a consumer exists. `static_mesh.describe` -- the
read that exists to tell you about a mesh -- says nothing about who is drawing it, so the only way to
discover the situation is to attempt the rebuild.

**Fix:** report live render consumers from `static_mesh.describe`. The scan already exists in
`Handlers/Model/MeshRebuildRenderGuard.h`; this is exposing it on a read. That is the third item of
the parent ticket's suggested-fix list, left undone because it is outside the geometry module the fix
lived in.

## History
- `#1-third-fix-item-not-taken` `OPEN` reporter -- Recorded by the agent that added the rebuild guard.
- `#2-scan-lifted-to-main-module` `IN-REVIEW` developer -- "Lifted the guard's scan half (candidate class table, live-consumer walk, target-loaded probe) out of PinWrightGeometry's MeshRebuildRenderGuard.h into Source/PinWright/Private/Utils/MeshRenderConsumerScan.h so the main module can read it without a PinWright -> PinWrightGeometry dependency; StaticMeshDescribeHandler.cpp now returns rebuildRenderConsumers {scannedClasses, components} from that same scan, and PinWright.static_mesh.describe.ReportsLiveRenderConsumers pins the report against the guard's own walk."
