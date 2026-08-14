---
id: B-copy-weights-duplicates-profile-info
title: "skeleton.copy_weights appends a duplicate FSkinWeightProfileInfo on every re-run — the mesh's profile list grows unboundedly"
status: IN-REVIEW
severity: Medium
category: bug
tags: [skeleton, skin-weights, copy-weights, idempotency, asset-bloat, duplicate-entry]
---

# copy_weights appends a duplicate profile entry on every re-run

`skeleton.copy_weights` registers its skin-weight profile on the target mesh with an
**unconditional** `AddSkinWeightProfile`. Run the verb twice with the same `profileName` and the
mesh's `SkinWeightProfiles` array carries the name twice; run it N times and it carries it N times.

Line numbers as read at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`
(`Plugins/PinWright/`).

`Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp:753-755`:

```cpp
    FSkinWeightProfileInfo NewProfile;
    NewProfile.Name = FName(*ProfileName);
    TargetMesh->AddSkinWeightProfile(NewProfile);
```

No existence check. The engine does not dedupe on its behalf —
`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\SkeletalMesh.cpp:6712-6718` is a bare
`SkinWeightProfiles.Add(Profile);`.

The plugin's own shared helper already knows this: `WriteSkinWeightProfile` guards the identical
call with a `ContainsByPredicate` check at
`Source/PinWright/Private/Handlers/Animation/SkinWeightTransferUtils.h:479-486`, precisely because
`AddSkinWeightProfile` appends blindly. `copy_weights` is the one weight verb that does not use
that helper — it keeps a private copy of the profile-persist logic (`SkeletalMeshHandler.cpp:753-767`)
and the guard is the part it did not copy.

## Impact

Re-running a weight port is a normal thing to do — after tweaking the source mesh, after a failed
attempt, or automatically: the transport timeout is response-only, so a timed-out `copy_weights`
keeps running and commits while the client legitimately retries. The board's own handler convention
is explicit that *"mutating handlers must be idempotent under client retry (upsert, not append)"*
(`agent-conventions.md`, Handler + dispatcher patterns). This one appends.

Consequences of the duplicate entries: the saved `.uasset` grows on each run, profile enumeration
(`skeleton.describe_skin_weights`, `USkeletalMesh::GetSkinWeightProfiles()`, the Skin Weight Profile
editor UI) reports the same name several times, and any consumer that resolves a profile by
iterating the array now depends on which duplicate it hits first. No error, no warning, and the
verb reports success with a normal `verticesCopied` count.

## Fix

Delete the inline block and have `copy_weights` call
`SkinWeightTransferUtils::WriteSkinWeightProfile` like the other three mutators do — it already
performs the guarded `AddSkinWeightProfile`, the `SkinWeightProfiles.FindOrAdd`, and the
`RebuildSourceModelInfluences` rebuild. Re-guarding the inline copy in place would also work but
leaves the duplicated persist logic, which is a live drift hazard in its own right: the same
duplication is why `copy_weights` will silently miss the index-space fix tracked in
`B-skin-weight-transfer-writes-section-local-bone-indices`. One edit closes both.

Regression test: run `copy_weights` twice with the same `profileName` on a fixture and assert
`GetSkinWeightProfiles().Num()` is 1, not 2. Fails against the current code.

severity rationale: impact=silent wrong asset state on a normal, retry-reachable path (duplicate
entries persisted into a saved `.uasset`, non-idempotent against the documented client-retry
contract) — short of corruption, since the weight data itself is intact × reach=rare (skin-weight
authoring) -> Medium

## History
- `#1-triage-unconditional-add` `OPEN` reporter — Found during a mesh/skeletal authoring triage at plugin HEAD `9da255f6d0ef5017cfddee03cd9458266c99d764`. `skeleton.copy_weights` calls `TargetMesh->AddSkinWeightProfile(NewProfile)` unconditionally at `SkeletalMeshHandler.cpp:753-755`; `USkeletalMesh::AddSkinWeightProfile` is a bare `SkinWeightProfiles.Add(Profile)` with no dedupe (`UE_5.8 .../SkeletalMesh.cpp:6712-6718`), so every re-run with the same `profileName` appends another `FSkinWeightProfileInfo`. The shared helper `WriteSkinWeightProfile` guards the same call with `ContainsByPredicate` (`SkinWeightTransferUtils.h:479-486`) — `copy_weights` is the only weight verb that bypasses that helper (private persist logic at `:753-767`) and it omitted the guard. Non-idempotent under the documented client-retry contract (transport timeout is response-only). Fix: call `WriteSkinWeightProfile` instead of the inline block, which also removes the drift that will make `copy_weights` miss the fix for `B-skin-weight-transfer-writes-section-local-bone-indices`. Not previously on the board.
- `#2-route-through-shared-persist` `IN-REVIEW` developer — Fixed by deleting the duplication rather than re-adding the guard. `skeleton.copy_weights` no longer calls `AddSkinWeightProfile` / `SkinWeightProfiles.FindOrAdd` / `RebuildSourceModelInfluences` itself; it builds the copied weights into a local array and calls `SkinWeightTransferUtils::WriteSkinWeightProfile`, which already guards the add with a `ContainsByPredicate` check. That is the shape `B-skin-weight-transfer-writes-section-local-bone-indices:62-64` recommended, and it closes both tickets at once: the duplicate entry, and the second `RebuildSourceModelInfluences` call site that a shared-helper fix would have silently missed. Re-running with the same `profileName` now overwrites that profile instead of registering a second one, which the verb summary now states. Reinforcement: `RebuildSourceModelInfluences` gained a REQUIRED leading `const FSkeletalMeshLODModel&` parameter, so any future attempt to re-open a private persist path fails to compile. **NOT COMPILED, NOT RUN.**
