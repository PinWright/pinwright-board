---
id: B-geometry-target-label-collision-wrong-actor
title: "GeometryTarget resolves only the first matching non-unique actor label, so geometry mutators can edit the wrong mesh and cannot address the intended duplicate"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, actor-resolution, duplicate-label, wrong-target, destructive-mutation, ambiguity, silent-success]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Colliding DynamicMeshActor labels route geometry operations to an arbitrary actor

`GeometryTarget.cpp:23-36` iterates `ADynamicMeshActor` objects, compares only
`GetActorLabel()`, and returns the first match. `FindAnyActor` repeats the same rule at `:39-52`.
The shared `ResolveOrSendError` routes virtually every geometry read and mutator through that helper.
It does not accept an internal object name or object path and does not collect matches to report
ambiguity.

The premise is explicitly guaranteed by UE, not hypothetical: `ActorEditor.cpp:1280-1282` says
actor labels are not supposed to be unique, and `AActor::SetActorLabel` stores the requested label
without uniquing it. `GeometryTarget::Spawn` calls exactly that setter at `:159`, so creating two
geometry actors with the same requested name is sufficient to make the state reachable inside this
module. A later destructive call such as `geometry.delete_triangle`, `boolean_subtract`, or
`convert_to_static_mesh` can succeed against the first actor while the intended actor is untouched;
there is no accepted identifier that can select the second one.

## What should happen

Adopt a deterministic two-pass resolver: exact object path/internal object name first; otherwise
collect display-label matches and refuse with an ambiguity error when more than one exists. Return
the chosen actor's real object path/name in every mutator response. Apply the rule to both
`FindMeshActor` and `FindAnyActor`, including tool/spline operands.

**Workaround:** keep all DynamicMeshActor display labels unique before using `geometry.*`. Unlike
`actor.*`, passing the unique object-name leaf is not a workaround because this helper compares
labels only.

## Related

`E-actor-name-resolution-label-collision` documents how `actor.*` callers can use the unique object
name. It does not cover this separate resolver, where that collision-safe identifier is rejected.

## Fix

TRUE. `GeometryTarget` used first-hit display-label scans, while direct loft, sweep, spline,
boolean-trim, and asset-reuse callers could discard the resolver's ambiguity distinction. The fix
adds resolver policies: geometry accepts exact object path, exact internal object name, then exact
display label, with no label-substring tier; DynamicMesh label candidates are class-filtered, while
an exact non-mesh path/name is treated as the explicitly requested identity. `reuseExisting` uses
an exact-label-only policy so a create label cannot be stolen by another actor's path or name.
Every ambiguity uses the typed `AMBIGUOUS_ACTOR_NAME` payload with `matchedBy`, a bounded-to-emitted
`candidateCount`, and candidate label/name/path/class records; object-name ambiguity recommends full
paths, while label ambiguity recommends unique names or paths. Resolver output is cleared before
failure, source/target errors precede spline errors, and existing-target mutators preserve the
request echo while adding canonical resolved path/name fields qualified as needed (`target`,
`tool`, `trim`, `source`, and `spline`). Static-mesh source-actor loads now retain the resolved
source pointer and emit `sourceActorPath`/`sourceActorObjectName`; the wiki response shapes name
those fields. No parseable GUID input contract exists in the plugin, so GUID matching was not
fabricated.

Files changed: `Source/PinWright/Private/Utils/ActorUtils.h`,
`Source/PinWright/Private/Utils/ActorUtils.cpp`,
`Source/PinWright/Private/Handlers/Actor/ActorNameParamUtils.h`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryTarget.h`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryTarget.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryUtils.h`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryUtils.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/BooleanHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/BulkEditHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryTransformHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/LODCollisionHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshMeasureHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshInfoHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshOpsHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshIOHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/MeshAssetIOHandler.cpp`,
`Source/PinWrightGeometry/Private/Handlers/Geometry/SkeletalMeshAssetIOHandler.cpp`,
`Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometryTargetResolution.cpp`, and
`Docs/wiki-src/geometry.md`.

Test id: `PinWright.geometry.target.DuplicateLabelReturnsAmbiguousCandidates` (added; not run by
the source-only constraint). It covers duplicate DynamicMesh labels, a same-label non-mesh actor,
substring rejection, nullable-wrapper refusal, structured ambiguity fields/count, exact name/path
selection, a real mutator identity response, and boolean-trim error ordering. Deliberately not
changed: GUID input support without an established contract, global actor substring behavior,
global candidate-array bounding, the small shared-resolver duplicate-scan optimization, unrelated
handlers, outer repositories, engine, Config/Saved, commits, and runtime/editor/build state.

## History
- `#1-source-pattern-scan` `OPEN` reporter — Both geometry resolvers return the first exact display-label match; UE documents labels as non-unique and the module's own spawn helper preserves duplicate labels. Source-only; no actor was created or mutated.
- `#2-geometry-target-resolution` `IN-REVIEW` developer — Verified TRUE and routed geometry actor targets through deterministic path/name/label resolution with typed ambiguity candidates; added the regression test and updated the geometry identifier contract.
