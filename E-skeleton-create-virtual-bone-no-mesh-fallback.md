---
id: E-skeleton-create-virtual-bone-no-mesh-fallback
title: "skeleton.create_virtual_bone / delete_virtual_bone are the only mesh-relevant skeleton methods that reject skeletalMeshPath — they force a separate _Skeleton path resolution mid-workflow"
status: OPEN
severity: Low
category: ergonomic
tags: [skeleton, virtual-bone, param-consistency, path-resolver, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# skeleton.create_virtual_bone has no skeletalMeshPath fallback, unlike every sibling method

Across the `skeleton.*` namespace, nearly every mesh-relevant method accepts
**either** `skeletonPath` **or** `skeletalMeshPath`, resolving the latter to the
mesh's bound Skeleton via `Ctx.GetStringFirstOf({skeletonPath, skeletalMeshPath})`
plus the combined `LoadSkeletonFromPathSkelOrMesh`-style resolver. This lets a
caller drive an entire rigging workflow off the one `SK_*` mesh path they already
have. Methods that do this (verified in `SkeletonHandler.cpp`): `get_info` (:104-109),
`list_bones` (:140-144), `describe_mesh` (:198-205), `list_sockets` (:900-904),
`create_socket` (:961-971), `configure_socket` (:1039-1049), `delete_socket`
(:1122-1128), `list_virtual_bones` (:776-781).

`skeleton.create_virtual_bone` (:701-728) and its counterpart
`skeleton.delete_virtual_bone` (:837-854) **break that contract**: they register
`skeletonPath` as `RPC_PARAM_REQ` with **no** `skeletalMeshPath` param at all, and
call the **bare** `LoadSkeletonFromPathSkel(SkeletonPath, ...)` (:728), which only
`StaticLoadObject(USkeleton::StaticClass(), ...)` — it does **not** fall back to
loading the path as a `USkeletalMesh` and reading `Mesh->GetSkeleton()`. So even if
a caller passed the mesh path *into* `skeletonPath`, it would be rejected as
"Asset is not a skeleton". These are the only two mesh-relevant skeleton methods
with this asymmetry.

## Impact (PROCESS friction)

In a mesh-path-driven workflow (build sockets, read info, list bones — all off the
one `SK_DinoDragon` mesh path), `create_virtual_bone` is the single call that
forces the caller to **break stride**: resolve the bound `_Skeleton` asset path
first (from a prior `describe_mesh`/`get_info` reading), then pass that explicit
skeleton path only here. It is a discoverability + consistency tax — the author of
the friction task spotted it only because they had already read the wiki and
noticed this one method's param list differed, then pre-resolved the skeleton path
to avoid a failed call. A caller who hadn't read every page carefully would pass
the mesh path (as they had for the previous 8 calls) and hit a confusing
`SKELETON_NOT_FOUND` on an asset they know exists.

## What it should do

- Register `skeletalMeshPath` as an optional fallback on both
  `skeleton.create_virtual_bone` and `skeleton.delete_virtual_bone`, and resolve
  via `Ctx.GetStringFirstOf({skeletonPath, skeletalMeshPath})` + the
  mesh-aware resolver (the same combined helper the sibling methods use), so a
  mesh path resolves to its bound Skeleton — matching `create_socket` /
  `list_virtual_bones`.
- **Docs (`docs/wiki-src/skeleton.md`):** until the param is added, the
  `### skeleton.create_virtual_bone` / `### skeleton.delete_virtual_bone`
  sections should explicitly call out that **only** `skeletonPath` is accepted
  here (no `skeletalMeshPath` fallback), and how to obtain the bound `_Skeleton`
  path (from `describe_mesh` / `get_info`), so the asymmetry is discoverable.

## Distinct from

- `E-skeleton-list-bones-no-limit-spills` (OPEN) — same namespace but a
  response-size/projection gap on `list_bones`, not a path-parameter inconsistency.
- `F-skeleton-no-mesh-for-physics-asset` (OPEN) — about a *bare authored skeleton
  having no bound mesh* for physics-asset creation; this ticket is the inverse
  (a real mesh exists, but this one method won't accept its path).

## History
- `#1-initial-audit` `OPEN` reporter — Filed by the struggle auditor from a clean
  SK_DinoDragon rigging task (namespace `skeleton`, outcome clean, 10 calls, all
  ok). Friction note: "create_virtual_bone is the only skeleton method whose wiki
  lists skeletonPath as required with no skeletalMeshPath fallback, so I used the
  explicit _Skeleton path there (resolved from describe_mesh), which worked first
  try." Confirmed in source: `SkeletonHandler.cpp` registers
  `create_virtual_bone` (:704) and `delete_virtual_bone` (:840) with
  `skeletonPath` as the only path param (`RPC_PARAM_REQ`, no `skeletalMeshPath`),
  and they call the bare `LoadSkeletonFromPathSkel` (:728) that loads only a
  `USkeleton` — no mesh→skeleton fallback. Every other mesh-relevant skeleton
  method in the same task (`get_info`, `list_bones`, `describe_mesh`,
  `create_socket`, `configure_socket`, `list_sockets`, `list_virtual_bones`)
  accepts both via `GetStringFirstOf({skeletonPath, skeletalMeshPath})`. Proposed:
  add the `skeletalMeshPath` fallback to both methods for namespace consistency,
  and meanwhile document the asymmetry in the `### skeleton.create_virtual_bone` /
  `### skeleton.delete_virtual_bone` sections of `docs/wiki-src/skeleton.md`.
