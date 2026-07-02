---
id: E-get-physics-asset-info-doc-fields-mismatch
title: "skeleton.get_physics_asset_info's doc promises a 'summary' with 'total primitive count' + 'bound SkeletalMesh path', but the result omits both named fields and instead dumps the FULL bodies[]+constraints[] arrays (22KB, larger than list_physics_bodies, overflows the 10k spill threshold)"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [skeleton, physics-asset, get_physics_asset_info, misleading-doc, readback, response-size, oversized, summary]
---

# `skeleton.get_physics_asset_info`'s documented "summary" fields are absent, and it dumps full arrays instead

`skeleton.get_physics_asset_info`'s wiki page describes the verb, verbatim
(`Saved/PinWright/wiki/skeleton.get_physics_asset_info.md:7`):

> Read summary metadata about a UPhysicsAsset: body count, constraint count,
> **total primitive count**, plus the **bound SkeletalMesh path** if the asset
> is referenced. Read-only counterpart to skeleton.list_physics_bodies.

Three of those promises do not hold against the actual output. The result's
top-level fields are exactly `physicsAssetPath`, `name`, `numBodies`,
`numConstraints`, `bodies[]`, `constraints[]` — nothing else:

1. **"total primitive count" — absent.** There is no scalar total-primitive
   field anywhere in the response. The only primitive counts are the per-body
   `numSpheres`/`numBoxes`/`numCapsules`/`numConvex` inside each element of the
   `bodies[]` array; a caller wanting the documented "total primitive count"
   must iterate and sum them itself.
2. **"bound SkeletalMesh path" — absent.** No `skeletalMeshPath` / `boundMesh`
   field is ever emitted, even when the verb was *called with* a
   `skeletalMeshPath` and auto-found the asset from it. The doc says it returns
   "the bound SkeletalMesh path if the asset is referenced"; it never does.
3. **It is not a "summary" / "read-only counterpart" — it is a SUPERSET.** Far
   from a lightweight summary, the verb serializes the **full `bodies[]` array**
   (one object per body — duplicating exactly what `skeleton.list_physics_bodies`
   returns) **plus the full `constraints[]` array** (one object per constraint,
   with `name`/`bone1`/`bone2`). On `SK_Spider_PhysicsAsset` (36 bodies, 34
   constraints) this is **22368 chars** — *larger* than
   `list_physics_bodies` on the same asset (**16938 chars**) — so the
   "summary" verb **overflows the 10000-char inline spill threshold** and is
   written to a `Saved/PinWright/HttpResponses/.../<uuid>.json` file, forcing a
   Read of the spilled payload, while the detail verb it claims to summarize
   *also* spills. A summary that is bigger than the detail it summarizes, and
   that overflows where the user only asked for "body count, constraint count,
   total primitive count."

## Why this is concretely misleading (not just a missing nicety)

The audited task ("audit the SK_Spider physics setup ... Tell me the body
count, constraint count, and **total primitive count**") asked for precisely the
three scalars the doc advertises as the summary. The doc led the agent to expect
a small summary object carrying `total primitive count` and the `bound
SkeletalMesh path`; instead it got a 22KB full-array dump that overflowed inline
and spilled to a file, with neither named field present — the body/constraint
counts had to be read from `numBodies`/`numConstraints` and the "total primitive
count" hand-summed from the `bodies[]` array the doc never says is returned.
This is the same misleading-doc family as `E-asset-get-doc-promises-tags`
(IN-REVIEW — `asset.get`'s "Read summary metadata ... asset registry tags" that
the result omits): a "Read summary metadata" verb whose documented field set
contradicts its real output. Here it is doubly misleading — two named fields are
missing *and* the "summary" framing is inverted (it returns more than the detail
verb, not less).

## What it should do

Pick one of the two arms used for `E-asset-get-doc-promises-tags`:

- **Behavior fix (makes the doc true, preferred):** have
  `get_physics_asset_info` emit the documented summary fields and drop (or gate
  behind an opt-in `verbose`/`includeBodies` flag) the full `bodies[]` /
  `constraints[]` arrays so the default payload is the promised lightweight
  summary that stays inline:
  - add a scalar `numPrimitives` (sum of all bodies' sphere+box+capsule+convex
    counts) — the "total primitive count" the doc names;
  - add `skeletalMeshPath` (the bound `USkeletalMesh`, when the asset is
    referenced / when the verb was called with one) — the "bound SkeletalMesh
    path" the doc names;
  - keep `numBodies` / `numConstraints`; move the per-body/per-constraint arrays
    behind an opt-in flag (default off) so the summary verb is actually a
    summary and the full enumeration stays the job of
    `skeleton.list_physics_bodies`.
- **Doc fix (alternative):** the false "summary" text is the C++
  `REGISTER_RPC_HANDLER` summary string at
  `Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp` (the
  `skeleton.get_physics_asset_info` registration line) — that string is the
  single source of truth the generated wiki page
  (`Saved/PinWright/wiki/skeleton.get_physics_asset_info.md:7`) renders from.
  `docs/wiki-src/skeleton.md` has **no** `### skeleton.get_physics_asset_info`
  overlay section today, so a doc fix would edit that C++ summary string (and
  optionally add an overlay H3). Rewrite it to describe what the verb actually
  returns — `numBodies`, `numConstraints`, and the full `bodies`/`constraints`
  arrays — strike "total primitive count" and "bound SkeletalMesh path", and
  note that the response can exceed the 10000-char inline budget and spill to a
  HttpResponses file on a typical ragdoll.

The behavior fix is preferred: it closes the misdirection at the verb the doc
points to and gives the summary verb a payload that doesn't overflow.

## Repro (verbatim, replay-confirmed)

Asset: `/Game/ExampleContent/IKRig/Mesh/Spider/SK_Spider_PhysicsAsset.SK_Spider_PhysicsAsset`
(36 bodies / 34 constraints, bound by `/Game/ExampleContent/IKRig/Mesh/Spider/SK_Spider.SK_Spider`).

```
call("skeleton.get_physics_asset_info",
     {skeletalMeshPath:"/Game/ExampleContent/IKRig/Mesh/Spider/SK_Spider.SK_Spider"})
→ outputTooLong: "Response exceeds display limit (22368 chars, threshold 10000);
   full payload written to .../HttpResponses/.../<uuid>.json"
   structuredContent top-level keys = {physicsAssetPath, name, numBodies(=36),
     numConstraints(=34), bodies[36], constraints[34]}
   — NO numPrimitives / total-primitive field, NO skeletalMeshPath/boundMesh field.

call("skeleton.list_physics_bodies",
     {physicsAssetPath:".../SK_Spider_PhysicsAsset.SK_Spider_PhysicsAsset"})
→ outputTooLong: "Response exceeds display limit (16938 chars, threshold 10000) ..."
```

The "summary" verb (22368 chars) is larger than the detail verb it claims to be
the "read-only counterpart" to (16938 chars); both overflow and spill to file.

## Distinct from

- `E-asset-get-doc-promises-tags` (IN-REVIEW) — same misleading-doc family ("Read
  summary metadata ... <field>" the result omits) but a different verb
  (`asset.get`, registry tags). This is `skeleton.get_physics_asset_info` with
  *two* missing named fields plus an inverted "summary" framing.
- `F-skeleton-no-mesh-for-physics-asset` (IN-REVIEW) — that is the *create* path
  (no mesh/preview-mesh to bind so `create_physics_asset` is unreachable from a
  bare skeleton); this is the *read* verb's doc-vs-output field mismatch and
  response size on an already-bound asset. Different defect, different verb.
- `E-volume-get-info-no-limit-spills` (OPEN) — also a verbose reader with no
  limit/projection that spills, but a different namespace
  (`volume.get_volumes_info`) and the *size* angle only; this ticket's core is
  the *documented-field mismatch* (the size/overflow is corroborating evidence
  that the "summary" framing is wrong).

## History
- `#1-initial-repro` `OPEN` reporter — Filed from the SK_Spider ragdoll-audit
  struggle task (seed `skeleton.get_physics_asset_info`, outcome clean — the
  verb worked, finding is the doc-vs-output ergonomic). Replay-confirmed on
  `/Game/ExampleContent/IKRig/Mesh/Spider/SK_Spider_PhysicsAsset`: the wiki
  (`skeleton.get_physics_asset_info.md:7`) advertises a *summary* with "total
  primitive count" and "bound SkeletalMesh path" and calls it the "read-only
  counterpart to skeleton.list_physics_bodies", but the actual result carries
  only `{physicsAssetPath, name, numBodies, numConstraints, bodies[], constraints[]}`
  — no total-primitive scalar, no bound-mesh path — and dumps the full
  bodies+constraints arrays (22368 chars, *larger* than `list_physics_bodies`'s
  16938 chars on the same asset), overflowing the 10000-char inline threshold and
  spilling to a HttpResponses file. Same misleading-"Read summary metadata"-doc
  family as `E-asset-get-doc-promises-tags`. Proposed: emit `numPrimitives` +
  `skeletalMeshPath` and gate the full arrays behind an opt-in flag (behavior fix,
  preferred), or rewrite the C++ `REGISTER_RPC_HANDLER` summary string for
  `skeleton.get_physics_asset_info` in `PhysicsAssetHandler.cpp` (the single
  source the generated wiki renders from — `docs/wiki-src/skeleton.md` carries no
  `### skeleton.get_physics_asset_info` overlay today) to match the real output
  (doc fix). Gated on the quotable doc-vs-output mismatch above.
- `#2-reword-and-behavior-fix` `IN-REVIEW` fuzz3 — REWORDED then implemented the
  **behavior fix** (the preferred arm, mirroring the landed `E-asset-get-doc-promises-tags`
  precedent). Reword: the "Doc fix" arm and History `#1` cited a `###
  skeleton.get_physics_asset_info` overlay in `docs/wiki-src/skeleton.md` that
  does not exist — the false "summary" text lives ONLY in the C++
  `REGISTER_RPC_HANDLER` summary string (the single source the generated wiki
  page renders from); corrected both citations to point at that string in
  `PhysicsAssetHandler.cpp`. Fix (code): `skeleton.get_physics_asset_info` now
  emits the two doc-promised fields — `numPrimitives` (sum of every body's
  sphere+box+capsule+convex shapes — the "total primitive count") and
  `skeletalMeshPath` (the bound preview mesh, resolved from the call's
  `skeletalMeshPath` or `GetPreviewMesh()` — the "bound SkeletalMesh path") — and
  gates the full per-body `bodies[]` / per-constraint `constraints[]` arrays
  behind a new `includeBodies` flag (alias `verbose`, default off) so the default
  payload is the promised lightweight summary that stays inline instead of a 22 KB
  superset that overflows the spill threshold. The registered summary string was
  rewritten to describe the real fields + the opt-in flag. Files:
  `Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp`
  (get_physics_asset_info body + summary string + new `includeBodies` param).
  Regression test:
  `Source/PinWright/Private/Tests/Gameplay/TestAnimationHandlers.cpp`
  (`FPhysicsGetAssetInfoSummaryFieldsTest`,
  `PinWright.skeleton.get_physics_asset_info.SummaryFields`) — builds a real
  UPhysicsAsset through the production authoring chain, adds a deterministic extra
  sphere primitive, binds a preview mesh, dispatches the real registered handler,
  and asserts `numPrimitives` equals the summed shape count, `skeletalMeshPath`
  equals the bound mesh path, the `bodies`/`constraints` arrays are ABSENT by
  default and PRESENT under `includeBodies=true`. Reverting the fix (dropping the
  scalars / always dumping the arrays) fails it.
