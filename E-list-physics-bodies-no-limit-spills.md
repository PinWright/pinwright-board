---
id: E-list-physics-bodies-no-limit-spills
title: "skeleton.list_physics_bodies has no limit/projection/bone-filter — a normal 35+ body ragdoll overflows the 10k spill threshold and forces a Read of the HttpResponses file just to audit bodies"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [skeleton, physics-asset, list_physics_bodies, response-size, oversized, pagination, projection, docs]
---

# skeleton.list_physics_bodies has no limit/projection — it dumps every body and spills to file

`skeleton.list_physics_bodies` enumerates **every** `USkeletalBodySetup` in a
`UPhysicsAsset` and returns the full array in one payload, with **no `limit`, no
bone-name filter, and no field projection**
(`Handlers/Animation/PhysicsAssetHandler.cpp:336` registers only the two resolver
params `physicsAssetPath` / `skeletalMeshPath`). On any ordinary creature/character
ragdoll (35+ bodies, each emitting `boneName` + `considerForBounds` +
`collisionType` + per-primitive `sphereCount`/`boxCount`/`capsuleCount`/etc.) the
serialized array crosses the **10000-char spill threshold**, so the response comes
back as `outputTooLong` and the full payload is written to
`Saved/PinWright/HttpResponses/.../<uuid>.json`, forcing the caller to
**Read/Grep the spilled file** just to inspect the bodies it asked to audit.

(`SkeletalBodySetups` is a flat array, so — like the canonical sibling
`E-actor-list-no-limit-spills` — it needs `limit` + projection + a name filter, **not**
a pagination cursor.)

This is the same no-narrowing-verbose-reader shape already ticketed for
`skeleton.list_bones` (`E-skeleton-list-bones-no-limit-spills`),
`volume.get_volumes_info` (`E-volume-get-info-no-limit-spills`), `actor.list`,
and `inspect.list_objects` — a different RPC, same gap and same proposed fix
family.

## What's wrong

`Handlers/Animation/PhysicsAssetHandler.cpp:336` registers only `physicsAssetPath` /
`skeletalMeshPath`. The body loops `for (USkeletalBodySetup* BodySetup :
PhysicsAsset->SkeletalBodySetups)` (:373) and unconditionally appends one object
per body — `boneName` (:378), `considerForBounds` (:379), `collisionType` (:382-389),
and the four per-primitive counts (`sphereCount` :391, `boxCount` :392, `capsuleCount`
:393, `convexCount` :394) — with no cap, no `boneName` substring filter, and no
`fields` gate, then emits the whole bodies array + `count` + `constraintCount`
(:400-402). There is no parameter a caller could pass to reduce the payload, so the
natural "audit the ragdoll's bodies" call always pays the full dump + spill-to-file +
extra Read tax even when the caller wants only a handful of leg/foot bones.

## Evidence

From the SK_Spider ragdoll-audit struggle task (focus
`skeleton.get_physics_asset_info`, namespace `skeleton`, outcome ergo — the verbs
all worked; this is the response-size/Read-tax angle). The task called
`skeleton.list_physics_bodies` **twice** — once on the before-state audit (step 3)
and once on the after-state verify (step 5) — on
`/Game/ExampleContent/IKRig/Mesh/Spider/SK_Spider_PhysicsAsset.SK_Spider_PhysicsAsset`
(35 then 36 bodies). The verb's payload was **16938 chars** (per the replay in the
sibling ticket `E-get-physics-asset-info-doc-fields-mismatch`), over the
10000-char inline threshold, so it was `outputTooLong` and spilled to a
HttpResponses JSON file. Friction note, verbatim: *"oversized responses spilled to
HttpResponses JSON files but that is the documented overflow path, read cleanly
with Read/Grep."* — i.e. the agent normalized the spill+Read as routine, which is
precisely the ergonomic cost: a body-audit intent on a single creature paid the
full dump + spill-to-file + extra Read, twice.

## What it should do

Mirror the narrowing levers landed on the canonical sibling
`E-actor-list-no-limit-spills` (`#4-reword-and-fix`, the already-implemented
reference version of this exact family) — **no pagination cursor** (a flat
`SkeletalBodySetups` array does not need one; the cursor was struck from
`E-actor-list` during its own reword as gold-plating):

- Add an optional `limit` with default `0` = **all** (so the default output stays
  byte-identical for callers that don't pass it) that truncates after filtering
  while `count` reports the returned rows, a `totalCount` always reports the
  untruncated match count, and a `truncated` flag flips true when rows were elided
  — the same **detectable-elision** contract `E-recorder-list-sessions-limit` and
  `E-graph-connections-pagination` landed (and not a silently-eliding small default).
- Add an optional `boneName`/`nameFilter` (substring) narrowing so the common "show
  me the bodies on the leg/foot bones" case returns just the matching rows inline
  instead of dumping all 35+.
- A `fields` / `namesOnly` projection so the common "which bones already have
  bodies?" case drops the per-primitive count fields (the bulk of the bytes),
  mirroring the `fields`/`namesOnly` allow-list `actor.list` ships.
- **Docs (`docs/wiki-src/skeleton.md`):** **add** a `### skeleton.list_physics_bodies`
  section (it does not exist yet) noting that the method returns *all* bodies, that
  the result exceeds the inline budget on any normal ragdoll (~30+ bodies), and that
  `limit` / `boneName` / `namesOnly` / `fields` keep an audit inline.

**Fix:** Add `limit` (default `0`=all; emit `totalCount` + `truncated` alongside the
existing `count`), a `boneName`/`nameFilter` substring filter, and a
`namesOnly`/`fields` per-body projection (allow-list of the seven per-body keys,
defaulting to all so unprojected output is unchanged) to the
`skeleton.list_physics_bodies` handler in
`Handlers/Animation/PhysicsAssetHandler.cpp`, and add the
`### skeleton.list_physics_bodies` section to `docs/wiki-src/skeleton.md`. No
pagination cursor.

## Distinct from

- `E-get-physics-asset-info-doc-fields-mismatch` (OPEN, the judge's filing for
  this same task) — that ticket's core is `get_physics_asset_info`'s *documented
  summary fields being absent* and the "summary" framing being inverted; its
  proposed fix **explicitly keeps `skeleton.list_physics_bodies` as the full
  enumeration verb** (gating the arrays in `get_physics_asset_info` behind an
  opt-in flag). This ticket is the *other* verb's own lack of narrowing — the gap
  that fix leaves untouched. It cites the 16938-char `list_physics_bodies` payload
  only as corroborating size evidence; making it narrowable is out of its scope.
- `E-skeleton-list-bones-no-limit-spills` (OPEN) — identical *shape* (no
  limit/projection/name-filter → spill → forced Read) but on `skeleton.list_bones`
  (the reference-skeleton bone enumeration), a different method/handler. Same fix
  family, different RPC.
- `E-volume-get-info-no-limit-spills` (OPEN) — same shape on
  `volume.get_volumes_info`. Same proposed fix family, different namespace.
- `E-http-response-spill` (DONE) — the *generic* server-side spill mechanism (the
  file-reference fallback itself); this ticket is that a *specific verbose reader*
  has no narrowing to stay under the threshold in the first place.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the SK_Spider ragdoll-audit
  struggle audit (focus `skeleton.get_physics_asset_info`, outcome ergo — all 14
  calls ok, no is_error, no retries). PROCESS angle distinct from the judge's
  `E-get-physics-asset-info-doc-fields-mismatch`: that ticket fixes
  `get_physics_asset_info`'s doc/summary mismatch and *keeps*
  `skeleton.list_physics_bodies` as the full-enumeration verb, leaving this verb's
  own no-narrowing gap unaddressed. `skeleton.list_physics_bodies`
  (`PhysicsAssetHandler.cpp:336`) registers only `physicsAssetPath`/`skeletalMeshPath`
  (no `limit`/pagination/`boneName`-filter/projection; loops all `SkeletalBodySetups`
  :373, serializing boneName + considerForBounds + collisionType + 4 per-primitive
  counts per body). On `SK_Spider_PhysicsAsset` (35→36 bodies) the payload is
  16938 chars — over the 10000-char inline threshold — returned as
  `outputTooLong` and written to a HttpResponses JSON file, forcing a Read/Grep; the
  task hit it **twice** (before-state audit step 3, after-state verify step 5).
  Friction note: *"oversized responses spilled to HttpResponses JSON files ... read
  cleanly with Read/Grep"* — the agent normalized the spill+Read tax. Proposed: add
  `limit` (default-inline, 0=all, with untruncated `count`) per
  `E-recorder-list-sessions-limit`/`E-graph-connections-pagination`, a
  `boneName`/`nameFilter` substring narrowing, optionally a `fields`/`namesOnly`
  projection, and document the all-bodies-overflows-inline behavior in the
  `### skeleton.list_physics_bodies` section of `docs/wiki-src/skeleton.md`.
  Distinct from `E-skeleton-list-bones-no-limit-spills` /
  `E-volume-get-info-no-limit-spills` (same shape, different RPC) and
  `E-http-response-spill` (DONE, the spill mechanism).
- `#2-reword-and-fix` `IN-REVIEW` developer — Validity review (correctness/adversarial/historian)
  confirmed the defect is real and present in current source and distinct from the
  sibling tickets. Reworded three stale framings the lenses flagged: (a) dropped the
  "no pagination cursor" gap — `SkeletalBodySetups` is a flat array; the canonical
  sibling `E-actor-list-no-limit-spills #4` struck the cursor as gold-plating, so this
  family takes `limit` + projection + name-filter, not a cursor; (b) corrected the
  `limit` ask from "default small enough to stay inline" (silently-eliding) to default
  `0`=all with `totalCount`/`truncated` (the detectable-elision contract the family
  landed, keeping unprojected output byte-identical); (c) corrected the source citation
  to `Handlers/Animation/PhysicsAssetHandler.cpp` (not a `Skeleton/` dir) and the spill
  path to `Saved/PinWright/HttpResponses/`. Implemented the fix:
  `skeleton.list_physics_bodies` now takes `boneName`/`nameFilter` (case-insensitive
  substring, snake_case aliases), `limit` (default `0`=all; emits the existing `count`
  plus a new untruncated `totalCount` and a `truncated` flag), and a
  `namesOnly`/`fields` per-body projection (allow-list of the seven per-body keys via
  `FHandlerContext::ReadFieldProjection`; default keeps all so unprojected output is
  byte-identical). Added the missing `### skeleton.list_physics_bodies` overlay section
  documenting the all-bodies-overflows-inline behavior + the four narrowing levers.
  Files: `Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp` (handler),
  `docs/wiki-src/skeleton.md` (docs). Regression test
  `FSkeletonListPhysicsBodiesNarrowingTest` (`PinWright.skeleton.list_physics_bodies.Narrowing`)
  added to `Source/PinWright/Private/Tests/Gameplay/TestPhysicsHandlers.cpp`: builds a real
  on-disk `UPhysicsAsset` with 3 named bodies, drives the production handler, and asserts
  `limit=2` caps rows at 2 while `totalCount`=3/`truncated`=true, `namesOnly` drops the
  primitive-count fields, and a `boneName="Foot"` filter returns only the matching body —
  all against the production handler, so reverting the limit/projection/filter plumbing
  fails it.
</content>
</invoke>
