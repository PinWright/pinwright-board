---
id: B-create-physics-asset-skeletonpath-alias-dead
title: "skeleton.create_physics_asset's documented 'skeletonPath' alias is dead — the handler implements it body-side (GetStringFirstOf) but never registers it as a schema alias, so ValidateHandlerParams rejects a skeletonPath-only payload with MISSING_REQUIRED_PARAM before the body runs, contradicting the verb's own wiki/summary"
status: IN-REVIEW
severity: Low
category: bug
tags: [documented-alias-rejected, skeleton, physics-asset, create_physics_asset, skeletonpath, skeletalmeshpath, param-alias, schema-validation, ragdoll]
encounters: 1
lastSeen: 2026-07-02T18:01:29.1212299+03:00
---

# `skeleton.create_physics_asset` rejects its own documented `skeletonPath` alias

The verb's wiki page and its C++ `REGISTER_RPC_HANDLER` summary both explicitly
promise that a bare-skeleton path may be supplied under a `skeletonPath` alias,
verbatim (`Saved/PinWright/wiki/skeleton.create_physics_asset.md:11`, identical
string in the C++ param spec):

> `skeletalMeshPath` (`string`, required): Asset path to a SkeletalMesh, OR a
> bare USkeleton path (**the 'skeletonPath' alias is also accepted**). ...

That alias is **dead**. Passing the target only under `skeletonPath` — exactly
what the doc says is accepted — hard-fails at the schema pre-validation layer,
before the handler body ever runs:

```
call("skeleton.create_physics_asset",
     {skeletonPath:"/Game/OracleReplay/SK_AliasProbe",
      outputPath:"/Game/OracleReplay/PHYS_AliasProbe", minBoneLength:5})
→ [MISSING_REQUIRED_PARAM] Missing required parameter 'skeletalMeshPath' (type: string)
```

Note the error carries NO "Accepted aliases:" suffix — confirming the dispatcher
sees `skeletalMeshPath` as the *only* accepted wire name for the slot.

## Root cause (ground truth from source)

The alias is implemented in the WRONG layer. The handler declares the target as
a bare **required** param and then reads it body-side with a two-key alias
getter:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/PhysicsAssetHandler.cpp:207`
  — `RPC_PARAM_REQ("skeletalMeshPath", "string", "... (the 'skeletonPath' alias is also accepted). ...")`
  declares the required param and documents the alias, but registers **no**
  schema alias — it is a bare `RPC_PARAM_REQ`, whose macro (`ParamSpec.h:31`)
  leaves the `FParamSpec.Aliases` list empty, rather than the alias-carrying
  spec that `ParamAliasUtils::MakeAliasParamSpec` builds.
- `PhysicsAssetHandler.cpp:212` —
  `FString SkeletalMeshPath = Ctx.GetStringFirstOf({TEXT("skeletalMeshPath"), TEXT("skeletonPath")});`
  the body-level alias that would accept `skeletonPath` — but it is **never
  reached** for a `skeletonPath`-only payload.

The dispatcher validates required params against the `FParamSpec` schema BEFORE
the handler body runs, and it only recognizes schema-registered names:

- `Plugins/PinWright/Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:83-105`
  — `ValidateHandlerParams` iterates each `bRequired` spec; when
  `PayloadHasParamOrAlias(Params, Spec)` is false it emits
  `Ctx.SendError(TEXT("MISSING_REQUIRED_PARAM"), Message)` and returns false.
- `RpcDispatcher.cpp:55-64` — `CollectParamNames(Spec)` (the single source of
  truth for accepted wire names) returns only `Spec.Name` + `Spec.Aliases` +
  `Spec.TypedAliases`. `skeletonPath` is in none of them, so it is not a valid
  wire name and the payload is rejected before the body's `GetStringFirstOf`
  ever executes.

So the body-side alias is shadowed by schema validation. Because the slot is
`RPC_PARAM_REQ` (unlike the many other `GetStringFirstOf` sites in the codebase,
which are on `RPC_PARAM_OPT` slots and therefore not gated by required-param
validation), the documented alias can never fire.

## What it should do

Register `skeletonPath` as a real dispatcher-level alias of the required
`skeletalMeshPath` spec (the `FParamSpec` alias machinery used by the landed
`E-blueprint-param-name-path-vs-assetpath #4` and its reuse across the
material/widget/property alias tickets), so `{skeletonPath:...}` validates at the
wire level and the existing body-side `GetStringFirstOf` resolves it. The body
already handles the value correctly (see the positive replay below) — only the
schema registration is missing. (Alternatively, if the alias is not wanted, strike
the "the 'skeletonPath' alias is also accepted" clause from both the wiki and the
C++ summary so the doc stops promising a dead input — but the alias fix is
preferred since the whole point of the bare-skeleton feature, per
`F-skeleton-no-mesh-for-physics-asset`, is that a `skeleton.create_skeleton`
author reaches for `skeletonPath`.)

## Why this is a bug, not mere alias-drift friction

This is the OPPOSITE of the `E-scs-blueprintpath-no-path-alias` /
`E-property-objectpath-no-assetpath-alias` drift family, where the tool never
documented the guessed spelling and the rejection is correct. Here the tool's own
wiki AND registered summary **advertise `skeletonPath` as accepted**, and the
handler body **implements it** — the caller supplied precisely the documented
input and got a hard error. That is a valid documented input wrongly rejected: a
tool contradicting its own contract, not a caller guessing wrong.

severity rationale: impact=soft-blocker-at-worst — a documented input is
rejected, but the canonical `skeletalMeshPath` (named in the same doc sentence
AND, verbatim, in the error message itself: "Missing required parameter
'skeletalMeshPath'") works as a non-silent one-retry self-correct, so a
workaround exists (this is NOT the "no workaround, task impossible" band that
would reach Medium/High) × reach=rare (physics-asset creation, not an
every-session path → bump down one) -> Low. It is still `category: bug` (a
tool contradicting its own advertised contract), just a low-cost one.

## Repro (verbatim, replay-confirmed at HEAD)

1. `skeleton.create_skeleton {path:"/Game/OracleReplay/SK_AliasProbe", rootBoneName:"pelvis"}`
   → `{skeletonPath:"/Game/OracleReplay/SK_AliasProbe.SK_AliasProbe", rootBoneName:"pelvis", boneCount:1}`
2. `skeleton.create_physics_asset {skeletonPath:"/Game/OracleReplay/SK_AliasProbe", outputPath:"/Game/OracleReplay/PHYS_AliasProbe", minBoneLength:5}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'skeletalMeshPath' (type: string)`
   (the documented alias, rejected — no "Accepted aliases:" suffix)
3. Same call with the canonical name works:
   `skeleton.create_physics_asset {skeletalMeshPath:"/Game/OracleReplay/SK_AliasProbe", outputPath:"/Game/OracleReplay/PHYS_AliasProbe", minBoneLength:5}`
   → `{physicsAssetPath:".../PHYS_AliasProbe.PHYS_AliasProbe", skeletonPath:".../SK_AliasProbe.SK_AliasProbe", bodyCount:1, constraintCount:0, fromSkeleton:true}`

Step 3 confirms the body-side alias handling is correct — the value resolves fine
once it reaches the body. Only the schema-level registration blocks step 2.

## Distinct from

- `F-skeleton-no-mesh-for-physics-asset` (IN-REVIEW) — the CAPABILITY GAP that
  added the bare-skeleton branch to `create_physics_asset` (its fix's `#2` also
  updated the docs to advertise `skeletonPath`, which is the very doc that now
  overpromises). That ticket's repro is `MESH_NOT_FOUND` at the mesh-load path;
  this is `MISSING_REQUIRED_PARAM` at schema pre-validation, a different layer and
  a different root cause (the alias is never schema-registered). The gap fix works
  only when the caller uses `skeletalMeshPath` for the skeleton path; it left the
  documented `skeletonPath` alias dead.
- `E-scs-blueprintpath-no-path-alias` / `E-property-objectpath-no-assetpath-alias`
  (OPEN) — alias-drift ergonomics where an UNdocumented guess is correctly
  rejected. Here the alias is DOCUMENTED and still rejected — a doc-contract bug,
  filed as `category: bug`.
- `E-cloth-verbs-wiki-overpromise-create-section` (IN-REVIEW) — a summary that
  oversells a workflow the handler NEVER implements (params `UNKNOWN_PARAMS`
  rejected). Here the handler DOES implement the alias body-side; it is only the
  schema registration that is missing.

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode finding from the
  quadruped-rig authoring task (touched skeleton.create_skeleton / add_bone /
  create_socket / create_virtual_bone / create_physics_asset / asset.save +
  readbacks). The attempt hit `[MISSING_REQUIRED_PARAM] Missing required parameter
  'skeletalMeshPath'` when passing `skeletonPath`, self-corrected with
  `skeletalMeshPath`. Replay-confirmed at HEAD on `/Game/OracleReplay/SK_AliasProbe`
  (see Repro): the documented `skeletonPath` alias is rejected at schema
  pre-validation because `PhysicsAssetHandler.cpp:207` declares a plain
  `RPC_PARAM_REQ("skeletalMeshPath", ...)` with no schema alias while the alias
  exists only body-side at `:212` (`GetStringFirstOf({skeletalMeshPath,
  skeletonPath})`), which `RpcDispatcher.cpp:83-105` `ValidateHandlerParams`
  (accepted names from `CollectParamNames`, `:55-64`) never reaches. The canonical
  `skeletalMeshPath` path succeeds (`fromSkeleton:true, bodyCount:1`), proving the
  body handles the value correctly. Fix: register `skeletonPath` as a dispatcher
  `FParamSpec` alias of the required `skeletalMeshPath` slot (or strike the alias
  clause from wiki + C++ summary). Severity Medium (documented-input-wrongly-
  rejected doc-contract violation on a rare path, one-retry self-correct).
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded severity Medium -> Low:
  per the board rubric, "rejecting valid input" only reaches Medium/High "with
  no workaround." Here the canonical `skeletalMeshPath` (named in the same doc
  sentence AND verbatim in the error message) works as a one-retry self-correct,
  so the base impact is soft-blocker-at-worst, and physics-asset creation is a
  rare edge path (bump down one) -> Low. Kept `category: bug` (the tool
  contradicts its own advertised contract). Also corrected a body imprecision
  (there is no `RPC_PARAM_REQ_ALIAS` macro; the real mechanism is
  `ParamAliasUtils::MakeAliasParamSpec`). Fix: registered `skeletonPath` as a
  schema alias of the required `skeletalMeshPath` slot by replacing the bare
  `RPC_PARAM_REQ` at `PhysicsAssetHandler.cpp:207` with
  `ParamAliasUtils::MakeAliasParamSpec(TEXT("skeletalMeshPath"), ..., true,
  {skeletalMeshPath, skeletonPath})` (+ include `Handlers/ParamAliasUtils.h`);
  the existing body-side `GetStringFirstOf` at `:212` then resolves it. The
  dispatcher now accepts a `skeletonPath`-only payload past `ValidateHandlerParams`
  instead of hard-failing `MISSING_REQUIRED_PARAM`. Regression test (new file
  `Tests/Gameplay/TestCreatePhysicsAssetSkeletonPathAlias.cpp`): (1) static —
  the registered `skeletalMeshPath` spec is required and carries the `skeletonPath`
  alias; (2) end-to-end — a `skeletonPath`-only payload routed through the real
  `FRpcDispatcher::ProcessRequest` is NOT rejected with `MISSING_REQUIRED_PARAM`
  (reaches the body, yielding `MESH_NOT_FOUND` on the bogus probe path). Reverting
  the alias fails both.
