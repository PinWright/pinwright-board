---
id: B-complex-collision-hullcount-echoes-unclamped-request
title: "geometry.generate_complex_collision clamps maxHullCount to 1-64 for the engine but returns the unclamped request as hullCount"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, collision, max-hull-count, clamp, request-echo, wrong-result, response-honesty]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# `hullCount` describes a value the collision generator did not receive

`LODCollisionHandler.cpp:65` documents `maxHullCount` as clamped to 1-64. The handler reads the raw
request at `:69`, passes `FMath::Clamp(MaxHullCount, 1, 64)` to
`CollisionOptions.MaxConvexHullsPerMesh` at `:77`, then emits the original `MaxHullCount` as
`hullCount` at `:87`.

For `maxHullCount:0`, the engine receives 1 and the successful response says `hullCount:0`; for 100,
the engine receives 64 and the response says 100. `shapeCount` is a useful measured output but is
not the same concept: decomposition may produce fewer shapes than its budget. The caller therefore
has no field that says which budget actually ran. This is the catalog's request-echo-not-result-
readback and coercion/domain-drift shape.

## What should happen

Compute one `EffectiveMaxHullCount`, use it for the engine call, and return it as `hullCount` (or as
`effectiveMaxHullCount` while retaining `requestedMaxHullCount`). Add a warning whenever clamping
changes the value. Cover both the lower and upper bound with a handler-level response test.

**Workaround:** send only values in 1-64 and use `shapeCount` as the actual produced-shape count.

## Fix

`LODCollisionHandler.cpp` now clamps once into `effectiveMaxHullCount`, passes that value to
Geometry Script, and reports `hullCount` from the component's built
`BodySetup.AggGeom.ConvexElems` instead of echoing the request. The response retains both requested
and effective budgets, always reports `clamped`, and adds `limit: 64` plus a warning when the
request was outside `1..64`. Added handler-harness coverage for both lower (`0 -> 1`) and upper
(`100 -> 64`) clamp responses, including measured convex-hull and total-shape counts. Updated the
geometry wiki contract. Static/source-only verification; no Unreal build, automation, or MCP run.

## History
- `#1-source-pattern-scan` `OPEN` reporter — The applied value is clamped at `LODCollisionHandler.cpp:77`, while the response at `:87` echoes the raw request. Source-only; no RPC was run.
- `#2-measured-hullcount-and-clamp` `IN-REVIEW` developer — Changed `LODCollisionHandler.cpp` to
  report measured built convex hulls, retain requested/effective budgets, and disclose clamp state,
  limit, and warning; added `PinWright.geometry.generate_complex_collision.ReportsMeasuredHullCountAndClamp`
  handler coverage and documented the response in `Docs/wiki-src/geometry.md`. Static/source-only;
  no Unreal build, automation, or MCP run.
