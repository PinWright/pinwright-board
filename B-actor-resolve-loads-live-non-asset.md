---
id: B-actor-resolve-loads-live-non-asset
title: "Actor resolver's asset fallback calls LoadAsset on a live non-asset object path, logging an engine LoadAsset Error"
status: IN-REVIEW
severity: Low
category: bug
tags: [actor, resolver, log-noise, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:14:57Z
---

# Resolver fallback trips `LoadAsset failed: ... is not a valid asset`

`McpActorUtils::ResolveActorFiltered` (`Source/PinWright/Private/Utils/ActorUtils.cpp`) ends with an
asset fallback gated on `UEditorAssetLibrary::DoesAssetExist(ActorName)`, on the assumption that the
probe is registry-only and silent. It is not only on-disk: for a path naming a **loaded** object the
asset registry builds asset data from the live object, so `DoesAssetExist` returns true for a live
non-asset (an actor in a loaded world that is not the current editor/PIE world, a component path).
`LoadAsset` then finds that same object, rejects it (`!IsAsset()`, `EditorAssetSubsystem.cpp:105-110`) and logs:

`LogEditorAssetSubsystem: Error: LoadAsset failed: '/Temp/PinWrightTests/PinWrightActorResolverOther_<guid>.PinWrightActorResolverOther_<guid>:PersistentLevel.BlockingVolume_0' is not a valid asset.`

The resolution result is unchanged (unresolved), so the only effect is an engine `Error:` line on every
such lookup through any `actor.*` verb, which reads like a real failure and fails any automation test
that does not declare it. Observed in `PinWright.actor_utils.ResolveActor.ExactPathLiveActorClasses`
(the wrong-world case), `Saved/Logs/pw_gapwave_full_offscreen2.log:10650`.

**Fix:** skip the fallback when the path already resolves (`FSoftObjectPath::ResolveObject`) to a loaded
object that is not a `UPackage` and not an asset: `LoadAsset` would return that exact object and refuse it.
External (OFPA) actors are assets (`AActor::IsAsset`) and still reach the fallback.

## History
- `#1-live-non-asset-load-error` `OPEN` reporter — Found while triaging tests newly exposed by the per-test `bSuppressLogErrors` reset: resolving an actor path from a second loaded world logged `LoadAsset failed: ... is not a valid asset` from the resolver's asset fallback.
- `#2-skip-live-non-asset` `IN-REVIEW` developer — Changed the fallback gate in `Utils/ActorUtils.cpp` to skip `DoesAssetExist`/`LoadAsset` when the path resolves to a loaded non-package non-asset object. Regression check: `PinWright.actor_utils.ResolveActor.ExactPathLiveActorClasses` (wrong-world case) fails on the engine error if the gate is reverted. Compile-checked only; needs a suite run.
