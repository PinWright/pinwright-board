---
id: F-retopo-secondary-params-ignored
title: "geometry.quadrangulate + geometry.remesh_voxel still silently ignore their secondary declared params (preserveFeatures / featureAngleThreshold / surfaceDistance) after the size-knob fix"
status: OPEN
severity: Low
category: feature
tags: [geometry, quadrangulate, remesh_voxel, ignored-param, silent-no-op, accept-and-ignore, placeholder-remesh]
encounters: 1
lastSeen: 2026-07-13T12:40:00+03:00
---

# quadrangulate/remesh_voxel accept secondary knobs they never apply

Split off from `B-remesh-size-param-ignored` when that ticket's fix wired the
**primary** density knob (`targetQuadSize` / `voxelSize`) into the uniform remesh
via `EGeometryScriptUniformRemeshTargetType::TargetEdgeLength`. The two retopo verbs
in `AdvancedMeshOpsHandler.cpp` still declare **secondary** parameters in their
ParamSpec that the handler body never reads, so a caller who sets them gets a silent
no-op (accept-and-silently-ignore) — the exact anti-pattern `agent-conventions.md`
forbids ("Never fake-success a feature the running engine can't do").

## Still-ignored params (verified in current source)

- `geometry.quadrangulate` (`AdvancedMeshOpsHandler.cpp`) declares
  `preserveFeatures` (boolean) and `featureAngleThreshold` (number) in its
  ParamSpec, but the handler body reads neither — passing `preserveFeatures:false`
  vs `true`, or any `featureAngleThreshold`, produces identical output. The uniform
  remesh (`ApplyUniformRemesh` in TargetEdgeLength mode) has no dihedral-angle
  feature-preservation input, so these knobs cannot be honored on the current code
  path.
- `geometry.remesh_voxel` (same file) declares `surfaceDistance` (number) in its
  ParamSpec, but the handler reads only `voxelSize` + `fillHoles`; `surfaceDistance`
  is never read at all — a no-op. There is no surface-distance / offset input to
  `ApplyUniformRemesh`; a real voxel path (`ApplyMeshSolidify` / morphology) would be
  needed to give it meaning.

## Repro

1. `geometry.create_torus {name:T, majorRadius:100, minorRadius:30}`.
2. `geometry.quadrangulate {actorName:T, targetQuadSize:50, preserveFeatures:false}`
   vs `{..., preserveFeatures:true}` → byte-identical output; the sharp-feature knob
   has zero effect. Same for `featureAngleThreshold`.
3. `geometry.remesh_voxel {actorName:T, voxelSize:10, surfaceDistance:5}` vs
   `{..., surfaceDistance:0}` → identical output; `surfaceDistance` is discarded.

## Acceptance

Stop silently accepting-and-ignoring these params. Either:

- **Honor them** on a code path that can (real feature-preserving remesh for
  `preserveFeatures`/`featureAngleThreshold`; a genuine voxel solidify/offset path for
  `surfaceDistance`), with a test asserting the param changes the output; OR
- **Drop them from the ParamSpec** so the dispatcher rejects them with
  `UNKNOWN_PARAMS` instead of silently swallowing them (the honest minimum — the caller
  learns the knob is unsupported rather than trusting a lie). If dropped, note the
  limitation in the `note`/response or the `geometry.md` wiki overlay.

Either way, a declared param must never be silently discarded. Lower priority than the
primary density defect (now fixed): the mainline retopo density knob works, so a task
is no longer blocked — these are secondary quality knobs, hence Low.

## History
- `#1-split-from-size-param-fix` `OPEN` developer — Split out of `B-remesh-size-param-ignored` when its fix wired the primary size knob (targetQuadSize/voxelSize) onto the uniform remesh's TargetEdgeLength. The secondary declared params — quadrangulate's `preserveFeatures`/`featureAngleThreshold` and remesh_voxel's `surfaceDistance` — remain declared-but-never-read in `AdvancedMeshOpsHandler.cpp`, an accept-and-silently-ignore the uniform-remesh path cannot honor. Tracks either honoring them on a capable path or dropping them from the ParamSpec so the dispatcher rejects them. Low: mainline density control is fixed, these are secondary knobs.
