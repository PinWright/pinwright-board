---
id: E-geometry-check-health-blind-to-self-intersection
title: "geometry.check_health cannot see the self-intersection the pwmodel health gate now reports"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [geometry, check_health, FMeshHealth, self-intersection, parity, missing-signal]
encounters: 1
lastSeen: 2026-08-28
---

# Two health surfaces, one of which now sees more than the other

`B-pwmodel-health-blind-to-interior-membrane` and `B-pwmodel-health-no-self-intersection` were fixed by
adding `MeasureMeshSelfIntersection` and publishing `health.selfIntersections` from `model.validate` /
`model.compile`.

`MeasureMeshHealth` / `IsHealthy()` were deliberately left alone, so `geometry.check_health` — and
`mesh.measure`, and the two other callers — still report the old field set and still call a solid with
a membrane through it healthy.

That leaves two health surfaces disagreeing about the same mesh, which is its own trap: a caller who
learned the gate has three terms from the `model.*` docs will assume `check_health` applies them.

**Why it was left:** the self-intersection measurement builds an AABB tree per connected component,
which is a different cost class from `MeasureMeshHealth`'s documented allocation-free walk, and that
walk runs per part. Adding it unconditionally would change the cost profile for four callers that did
not ask for it.

**Fix direction:** an opt-in parameter on `check_health`, or an explicit statement in its response that
self-intersection was not measured. An absent measurement that reads as a clean one is the thing to
avoid — the same rule the `model.*` fields follow, where the fields are absent rather than zeroed when
declined.

## History
- `#1-left-by-the-health-fix` `OPEN` reporter — Recorded by the agent that added the self-intersection
  signal, which scoped itself to the published `model.*` gate its two tickets named.
- `#2-check-health-field-set-parity` `IN-REVIEW` developer — "Added an opt-in `checkSelfIntersection` param to geometry.check_health in MeshMeasureHandler.cpp: it emits `selfIntersectionsMeasured` unconditionally and `selfIntersections` / `selfIntersectingComponents` / `selfIntersectionsTruncated` only when the measurement ran, so a declined answer is stated rather than absent or zeroed; also emitted the already-measured `unreferencedVertices`, so the verb now publishes every field the `model.*` health block does. Docs/pwmodel-format.md loses its not-emitted exception; Docs/wiki-src/geometry.md gains the fields and the cost rationale. Regression: three tests in TestMeshMeasureHandler.cpp, one comparing the check_health key set against a live model.validate `health` block."
