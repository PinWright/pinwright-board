---
id: B-create-morph-target-empty-not-persisted
title: "skeleton.create_morph_target returns success but never persists the empty morph (count 0, invisible to list/set/delete)"
status: IN-REVIEW
severity: High
category: bug
tags: [skeleton, morph-target, fake-success, silent-failure, create]
---

# skeleton.create_morph_target is a success-returning no-op for empty morphs

`skeleton.create_morph_target` replies with a success-shaped response
(`{morphTargetName, morphTargetCount}`) but the freshly created `UMorphTarget`
is **not actually added to the mesh**. Immediately after the call,
`morphTargetCount` is reported as `0`, `skeleton.list_morph_targets` still
returns an empty array, and every documented follow-up
(`skeleton.set_morph_target_deltas`, `skeleton.delete_morph_target`) fails with
`[MORPH_NOT_FOUND]`. A repeat create on the same name does **not** return
`alreadyExists=true` (contradicting the wiki's documented idempotency),
confirming the first create never persisted anything.

This makes the entire documented authoring workflow impossible: the wiki and the
`set_morph_target_deltas` description tell the caller to "Create an empty
UMorphTarget ... Populate deltas via skeleton.set_morph_target_deltas" — but the
empty create never persists, so there is never a morph to populate. The success
response with a real-looking `morphTargetName` actively misleads the caller into
believing the morph exists; the failure only surfaces on the next call.

## Root cause (source)

`Source/EditorAutomationRpcGateway/Private/Handlers/Animation/MorphTargetHandler.cpp`,
`skeleton.create_morph_target` handler (lines 98-110):

```cpp
UMorphTarget* NewMorphTarget = NewObject<UMorphTarget>(Mesh, FName(*MorphTargetName));
// ...
Mesh->RegisterMorphTarget(NewMorphTarget);
McpSafeAssetSave(Mesh);
// ...
Result->SetNumberField(TEXT("morphTargetCount"), Mesh->GetMorphTargets().Num());  // -> 0
```

The morph target is created empty (no LOD models / deltas — by design, since
deltas are meant to be added later via `set_morph_target_deltas`). But the engine
drops empty morphs during registration. In
`Engine/Private/SkeletalMesh.cpp`, `USkeletalMesh::RegisterMorphTarget`
(UE 5.7, lines 4616-4658):

```cpp
// if the input morphtarget doesn't have valid data, do not add to the base morphtarget
ensureMsgf(MorphTarget->HasValidData(), TEXT("RegisterMorphTarget: %s has empty data."), ...);
// ...
RegisteredMorphTargets.Add( MorphTarget );
// ...
if (bRegistered && bInvalidateRenderData) { InitMorphTargetsAndRebuildRenderData(); }  // rebuilds, discarding the empty morph
```

The morph with no valid data does not survive `RegisterMorphTarget` /
`InitMorphTargetsAndRebuildRenderData` — hence `GetMorphTargets().Num()` is `0`
on the very next line and the morph is never reachable. The handler treats the
no-op as success.

## Why it matters

A caller doing the obvious round-trip — `create_morph_target` (empty) ->
`list_morph_targets` (verify) -> `set_morph_target_deltas` (populate) — is dead
on arrival at step 2: the list is unchanged, and step 3 returns
`[MORPH_NOT_FOUND]`. There is no order of operations that works: `set_deltas`
requires the morph to already exist, and `create` is the only way to make it
exist, but `create` of an empty morph never persists. This violates the repo's
never-fake-success rule and breaks the documented create->populate workflow.

## Fix options (any of)

1. **Make create populate valid data so registration succeeds.** Seed the new
   `UMorphTarget` with a minimal/zeroed LOD model (one delta) so
   `HasValidData()` is true and `RegisterMorphTarget` keeps it; the subsequent
   `set_morph_target_deltas` overwrites it. (Best: preserves the documented
   two-step empty-create-then-populate workflow.)
2. **Verify after register and report honestly.** After `RegisterMorphTarget`,
   re-check `FindMorphTarget` / `GetMorphTargets().Num()`; if the morph did not
   land, `SendError` instead of `SendSuccess` so the caller learns the create
   failed at the create, not on the next call.
3. **Merge create+populate.** Require deltas at create time (or accept optional
   deltas) so an empty morph is never registered.

Option 1 closes the workflow gap; option 2 is the minimum honesty fix.

## Repro

Mesh: `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon.SK_DinoDragon`
(85540 verts, 0 morph targets initially — verified via `describe_mesh` and
`list_morph_targets`).

1. `skeleton.list_morph_targets {skeletalMeshPath: ".../SK_DinoDragon.SK_DinoDragon"}`
   -> `{"morphTargets":[],"count":0}`  (baseline empty)
2. `skeleton.create_morph_target {skeletalMeshPath: "...", morphTargetName: "Jaw_Open"}`
   -> `{"morphTargetName":"Jaw_Open","morphTargetCount":0}`  (success, but count 0 — already a red flag)
3. `skeleton.list_morph_targets {...}` -> `{"morphTargets":[],"count":0}`  (Jaw_Open absent — never persisted)
4. `skeleton.create_morph_target {... "Jaw_Open"}` (repeat)
   -> `{"morphTargetName":"Jaw_Open","morphTargetCount":0}`  (NO `alreadyExists:true` — contradicts wiki idempotency claim)
5. `skeleton.set_morph_target_deltas {... "Jaw_Open", deltas:[{vertexIndex:0,positionDelta:{x:0,y:0,z:-5}}, ...]}`
   -> `[MORPH_NOT_FOUND] Morph target 'Jaw_Open' not found`
6. `skeleton.delete_morph_target {... "Jaw_Open"}`
   -> `[MORPH_NOT_FOUND] Morph target 'Jaw_Open' not found`

## History
- `#3-additional-misleading-error-text` `IN-REVIEW` reporter — Additional evidence (the `#2` fix works, but its new error text is self-contradicting). Replayed on `/MetaHumanCharacter/Face/SKM_Face.SKM_Face` (0 morphs baseline). `create_morph_target {morphTargetName: "FuzzSmile_Custom"}` (no deltas) now correctly errors instead of fake-succeeding — confirms the `#2` honesty fix landed. BUT the verbatim error is `[MORPH_NOT_PERSISTED] Morph target 'FuzzSmile_Custom' was not persisted: the engine drops empty morph targets (UMorphTarget::HasValidData() is false with no vertex deltas). Supply deltas at creation, or import a populated morph via skeleton.import_morph_targets.` The clause **"Supply deltas at creation"** is misleading/impossible: `create_morph_target` has no `deltas` param, so following that hint immediately fails — `create_morph_target {..., deltas:[{vertexIndex:0,positionDelta:{x:0,y:0,z:1.5}}]}` returns `[UNKNOWN_PARAMS] Unknown parameter(s) for 'skeleton.create_morph_target': [deltas]. Valid parameters: [skeletalMeshPath, morphTargetName].` The method's own remedy contradicts its own parameter list. Note the wiki page (`skeleton.create_morph_target.md` line 7) is already correct here — it says only "supply the shape via skeleton.import_morph_targets, or author it as an actual blendshape" with NO "supply deltas at creation" clause — so it is purely the runtime error string in `MorphTargetHandler.cpp` (the `SendError("MORPH_NOT_PERSISTED", ...)` message) that should drop the "Supply deltas at creation" half and point only to `skeleton.import_morph_targets` (matching the wiki), before this ticket closes. Seed was `skeleton.list_morph_targets` (works: returns count 0); culprit is the neighbor `skeleton.create_morph_target`.
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current fuzz2 source; no code written. `MorphTargetHandler.cpp` skeleton.create_morph_target now re-checks `Mesh->FindMorphTarget(...)` immediately after `RegisterMorphTarget` (lines 115-124) and `SendError("MORPH_NOT_PERSISTED", ...)` instead of fake-success when the engine evicts the empty morph; the success path (lines 126-132) is reached only when the morph actually persisted. The handler comment (107-114) and tool description (line 64) correctly attribute the eviction to `InitMorphTargets()` stripping `!HasValidData()` morphs (addressing the CORRECTNESS lens's reword of the original ticket's `ensureMsgf`/4627 mis-attribution). Regression test present: `FMorphCreateEmptyNotPersistedTest` (`Tests/Gameplay/TestAnimationHandlers.cpp:1652-1692`, `skeleton.create_morph_target.EmptyMorphNotFakeSuccess`) builds a transient `USkeletalMesh`, invokes the real handler via `InvokeHandlerWithCapture`, and asserts `!bSuccess`, `ErrorCode == "MORPH_NOT_PERSISTED"`, and `GetMorphTargets().Num() == 0` (would fail if the fix were reverted). Status left IN-REVIEW for a tester to verify the green build.
- `#1-initial-repro` `OPEN` reporter — Replayed against the live editor on `SK_DinoDragon` (0 morphs baseline). `create_morph_target "Jaw_Open"` returned success with `morphTargetCount:0`; `list_morph_targets` immediately after still `count:0` (Jaw_Open absent); a repeat `create` returned NO `alreadyExists:true` (contradicting wiki idempotency); `set_morph_target_deltas` and `delete_morph_target` both returned `[MORPH_NOT_FOUND]`. Root cause in `MorphTargetHandler.cpp:98-110`: the handler creates an EMPTY `UMorphTarget` and calls `Mesh->RegisterMorphTarget`, but `USkeletalMesh::RegisterMorphTarget` / `InitMorphTargetsAndRebuildRenderData` (`SkeletalMesh.cpp:4616-4658`) discard morphs with no valid data (`ensureMsgf(HasValidData())`, "do not add to the base morphtarget" comment). The handler reports the no-op as success — fake-success that breaks the documented create-empty->populate-deltas workflow. Seed was `skeleton.delete_morph_target`, but delete correctly errors (nothing was ever created); the bug is in the neighbor `skeleton.create_morph_target`. No prior board ticket references morph-target create.
