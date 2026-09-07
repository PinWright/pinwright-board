---
id: E-create-physics-asset-meshless-no-constraints-doc
title: "skeleton.create_physics_asset's summary promises 'collision capsules + joint constraints ... suitable for ragdoll', but on the meshless (bare-USkeleton) path it returns constraintCount:0 — bodies-only, no joint chain — with no doc warning, so callers expect a ready-to-flop ragdoll from one call"
status: OPEN
severity: Low
category: ergonomic
tags: [misleading-doc, docs, skeleton, physics-asset, create_physics_asset, ragdoll, constraints, meshless]
encounters: 1
costly: 1
lastSeen: 2026-07-13T08:31:50.8164083+03:00
---

# `skeleton.create_physics_asset`'s doc promises "joint constraints" + a ragdoll-suitable asset, but the meshless path yields bodies-only (constraintCount:0)

`skeleton.create_physics_asset` **works correctly** — this is a docs/discoverability
gap, not a tool bug. On the bare-`USkeleton` (meshless) path it deterministically
produces capsule bodies and **zero** joint constraints, which is intended behavior
(source-confirmed: `BuildBodiesFromSkeletonRefPose` in `PhysicsAssetHandler.cpp`
only appends `SkeletalBodySetups` and never touches `PhysicsAsset->ConstraintSetup`;
the response honestly reports `constraintCount:0`). The friction is that the verb's
wiki summary does **not** disclose this for the meshless path, so a caller reading
the doc expects a connected, ready-to-flop ragdoll from one call.

The generated page (`wiki-generated/skeleton.create_physics_asset.md:7`) opens,
verbatim:

> Create a new UPhysicsAsset (collision capsules + joint constraints) for a
> SkeletalMesh, suitable for ragdoll / cloth interaction. Auto-generates capsule
> bodies from bone reference poses; refine afterwards via skeleton.add_physics_body
> / configure_physics_body / add_physics_constraint.

Two things mislead on the meshless path:

1. The headline promises "**collision capsules + joint constraints**" and an asset
   "**suitable for ragdoll**". On a bare `USkeleton` you get capsules but **no
   constraints** — an unconnected pile of capsules that will not flop as one ragdoll
   until you hand-author the entire joint chain.
2. The only caveat is a generic trailing "refine afterwards via ...
   add_physics_constraint", which reads as **optional tuning**, not "on the meshless
   path you must build every parent-child constraint yourself." Nothing states that
   the meshless path returns `constraintCount:0`.

## Works but non-obvious — the process cost this imposed

On the quadruped-rig authoring task (build a from-scratch creature skeleton, then
"generate a basic ragdoll physics setup ... so it can flop over"), the doc led the
agent's plan to assume `create_physics_asset` on the bare `SK_Critter` would yield a
ready-to-flop ragdoll in one call — its plan had **no** constraint-authoring steps.
Reality: the call returned `bodyCount:10, constraintCount:0, fromSkeleton:true`, so
the agent had to read `skeleton.add_physics_constraint.md` (1 extra wiki read) and
issue **9** `skeleton.add_physics_constraint` calls (Root->hips, hips->spine_01,
spine_01->spine_02, spine_02->neck, neck->head, spine_02->leg_front_l/r,
hips->leg_back_l/r) to make the ragdoll actually connected. The agent's own friction
note: the raw auto-output "is not a connected ragdoll until you hand-author
constraints ... a user expecting a ready-to-flop ragdoll would be surprised."

(The mesh-backed path is different and NOT affected: `McpCreatePhysicsAssetFromSkeletalMeshHeadless`
routes through `FPhysicsAssetUtils::CreateFromSkeletalMesh`, which emits bodies AND
constraints, so the "collision capsules + joint constraints / suitable for ragdoll"
framing is accurate there. The doc's single summary just doesn't distinguish the two
paths.)

## What it should do (docs only)

Improve the `skeleton.create_physics_asset` doc so the meshless path is not a
surprise. Add a `### skeleton.create_physics_asset` overlay section to
`docs/wiki-src/skeleton.md` (there is none today), and/or reword the C++
`REGISTER_RPC_HANDLER` summary string in
`Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp` (the single
source the generated page renders from), to state plainly:

- The **meshless / bare-`USkeleton` (`fromSkeleton:true`) path yields bodies only —
  `constraintCount:0`** — and the caller must author the parent-child joint chain
  themselves via `skeleton.add_physics_constraint` before the asset behaves as a
  connected ragdoll.
- Reserve the "collision capsules + joint constraints" / "suitable for ragdoll"
  framing for the **mesh-backed** path (`FPhysicsAssetUtils::CreateFromSkeletalMesh`),
  which does emit both.

This closes the plan-divergence / expectation gap without changing behavior. (A
behavioral alternative — auto-generate the parent-child constraint chain from the
reference-skeleton parent links on the meshless path — is deliberately NOT proposed
here: `add_physics_constraint` is a documented, working path and the meshless
bodies-only output is intended per source, so the actionable gap is purely
documentation.)

severity rationale: impact=docs/discoverability (works but non-obvious; no defect —
honest response, intended behavior) × reach=rare (physics-asset creation from a
hand-authored meshless skeleton is a specialized ragdoll path, not an every-session
method) -> Low

## Distinct from

- `F-skeleton-no-mesh-for-physics-asset` (IN-REVIEW) — the CAPABILITY gap whose fix
  ADDED the meshless bodies-only path in the first place. That ticket is "no mesh to
  bind, so `create_physics_asset` is unreachable"; it does not address the summary's
  constraints/ragdoll overpromise that its own bodies-only output now creates. Same
  method, different root cause (existence of the path vs. the doc for it).
- `B-create-physics-asset-skeletonpath-alias-dead` (IN-REVIEW) — same method, but the
  documented `skeletonPath` schema alias being rejected (`MISSING_REQUIRED_PARAM`).
  Different layer (schema validation) and different root cause.
- `E-get-physics-asset-info-doc-fields-mismatch` (IN-REVIEW) — same `misleading-doc`
  family, but the READ verb (`get_physics_asset_info`) whose documented summary
  fields are absent + oversized output. Different method, different specifics.
- Same `misleading-doc` family as `E-asset-get-doc-promises-tags`,
  `E-cloth-verbs-wiki-overpromise-create-section`,
  `E-mrq-run-jobs-doc-promises-per-job-exit-status`,
  `E-static-mesh-describe-doc-promises-nanite` — each stays method-scoped per the
  "over-broad prior ticket must not absorb a distinct friction" rule; none names
  `create_physics_asset`'s meshless constraint overpromise, so this is filed
  separately.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of an otherwise clean,
  no-retry quadruped-rig authoring task (build SK_Critter from scratch, then
  "generate a basic ragdoll ... so it can flop over"). The `create_physics_asset`
  wiki summary ("collision capsules + joint constraints ... suitable for ragdoll")
  led Prep's plan to assume a one-call ready-to-flop ragdoll (no constraint steps),
  but the meshless call returned `bodyCount:10, constraintCount:0, fromSkeleton:true`,
  forcing 1 extra wiki read (`add_physics_constraint.md`) + 9 `add_physics_constraint`
  calls to connect the joint chain. The per-finding Judge source-confirmed the
  meshless bodies-only output is INTENDED and the response HONEST (no defect —
  `BuildBodiesFromSkeletonRefPose` never touches `ConstraintSetup`), so this is filed
  as a pure docs-discoverability improvement ("works but non-obvious"), not a bug:
  the summary should warn that the meshless (`fromSkeleton`) path yields
  `constraintCount:0` and the caller must author the joint chain via
  `add_physics_constraint`, and reserve the "joint constraints / suitable for ragdoll"
  framing for the mesh-backed path. Page to improve: `docs/wiki-src/skeleton.md` (add
  a `### skeleton.create_physics_asset` overlay) and/or the C++ `REGISTER_RPC_HANDLER`
  summary in `PhysicsAssetHandler.cpp`. severity Low (docs/discoverability × rare
  ragdoll-authoring path).
