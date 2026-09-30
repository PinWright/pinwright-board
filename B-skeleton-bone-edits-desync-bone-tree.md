---
id: B-skeleton-bone-edits-desync-bone-tree
title: "skeleton.remove_bone / set_bone_parent reindex the reference skeleton but leave USkeleton::BoneTree (per-bone translation retarget modes) on the old indices"
status: OPEN
severity: High
category: bug
tags: [skeleton, bone-tree, retargeting, wrong-result, reproduce-first, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# Direct bone edits desync the Skeleton's BoneTree

**Reproduce first.** This is inferred from engine and plugin source; it has not been run live.

`skeleton.add_bone`, `skeleton.remove_bone` and `skeleton.set_bone_parent` edit the Skeleton asset
through `FReferenceSkeletonModifier(USkeleton*)` directly
(`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:780`, `:852`, `:912`, plugin HEAD
`2580e7f4`). None of them calls `Modify()`, opens a transaction, touches bound meshes, or updates
`USkeleton::BoneTree`.

Why that matters:
- `FReferenceSkeleton::Remove` shifts every later raw bone index down by one
  (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Animation/ReferenceSkeleton.cpp:100-145`), and
  `SetParent` rebuilds the hierarchy order (`:363+`).
- The modifier's destructor only calls `RebuildRefSkeleton` (`ReferenceSkeleton.cpp:15-18`).
- `USkeleton::BoneTree` is a parallel per-bone array holding each bone's `TranslationRetargetingMode`
  (`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/Animation/Skeleton.h:304`), read by index in
  `GetBoneTranslationRetargetingMode` (`:861-867`).

So after `remove_bone` (and likely after a reordering `set_bone_parent`) retarget modes appear to land
on the wrong bones, silently, with a success response. After `add_bone` the new bone has no entry and
falls back to `Animation` (harmless default). The `.pwskel` compile path already knows this is needed:
it resets `BoneTree` explicitly (`Source/PinWright/Private/PwSkel/PwSkelAssetCreate.cpp:560-578`).

Secondary: editing a Skeleton that meshes are bound to never touches those meshes, so a remove or
reparent may leave them incompatible, with nothing in the response saying so.

## Repro to run

1. Skeleton with bones `root > a > b > c`; set `c` to `Skeleton` translation retargeting, others `Animation`.
2. `skeleton.remove_bone {boneName: "a"}` (children reparented).
3. Read retargeting modes by name (asset.dump `skeleton.json` or reflection). Expected: `c` still `Skeleton`. Suspected: the `Skeleton` mode now sits on a different bone or on a stale trailing entry.

**Fix:** remap `BoneTree` by bone name around the modifier (capture name → mode before, rebuild after),
the same way the `.pwskel` path resets it; add `Modify()` and an `FScopedTransaction`. When meshes are
bound to the Skeleton, report the meshes the edit made incompatible (or refuse).

## Acceptance

- A test that fails before the fix and shows retarget modes follow bone names after remove and after reparent.
- A remove on a Skeleton with a bound mesh reports the effect on that mesh.

**Effort:** S.

**Related:** `F-skeleton-edit-mesh-bones` (mesh-side bone editing).

## History
- `#1-bone-tree-not-remapped` `OPEN` reporter — Found in the 2026-09-30 animation gap analysis by reading plugin source at `2580e7f4` and UE 5.8 engine source; not reproduced live — reproduce first. Severity High if confirmed (silent wrong data on a normal path); treat as Medium until reproduced.
