---
id: F-gameplay-tag-referencers
title: "No gameplay_tags.find_referencers, and gameplay_tags.remove on a referenced tag answers removed:false with no reason"
status: IN-REVIEW
severity: Medium
category: feature
tags: [gameplay-tags, remove, referencers, asset-registry, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# No tag-referencer lookup; referenced-tag remove fails without a reason

`gameplay_tags` has no way to ask which assets use a tag. The engine already answers it: a saved
package that stores an `FGameplayTag` records it as a `SearchableName` dependency
(`FGameplayTag::PostSerialize`), and `IAssetRegistry::GetReferencers(FAssetIdentifier(FGameplayTag::StaticStruct(), Tag), ..., EDependencyCategory::SearchableName)`
returns the packages. The editor's own tag delete runs that query.

`gameplay_tags.remove` calls `IGameplayTagsEditorModule::DeleteTagFromINI`, which refuses to delete a
referenced tag (`GameplayTagsEditorModule.cpp` `DeleteTagFromINIInternal`, ~682 on 5.8, ~584 on
5.3). The refusal reaches only an editor toast; the verb then returned success with
`removed:false` and no reason, the same shape as removing a tag that does not exist. The caller
cannot tell "already gone" from "blocked by assets" and has no verb to find the assets.

Engines: `GetReferencers(const FAssetIdentifier&, TArray<FAssetIdentifier>&, EDependencyCategory, ...)`,
`FAssetIdentifier(UObject*, FName)`, `RequestGameplayTagChildrenInDictionary` and
`FGameplayTagNode::IsExplicitTag` exist unchanged on UE 5.3 through 5.8. UE 5.3's engine loop checks
the leaf tag name for every implicit parent it would delete (5.4+ checks each name).

**Fix:** new batch verb `gameplay_tags.find_referencers {tags[]}`; `gameplay_tags.remove` runs the
engine's referencer check itself and returns `TAG_IN_USE` with the referencers, and gives every other
`removed:false` a `reason`.

## History
- `#1-no-tag-referencer-lookup` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis (plan task 9). No verb lists a tag's referencers, and `gameplay_tags.remove` on a referenced tag returns success `removed:false` with no reason because `DeleteTagFromINI` reports its refusal only as an editor notification. Severity Medium: the task is doable only by a source dive plus an asset-registry call through `python.execute`.
- `#2-find-referencers-and-remove-reason` `IN-REVIEW` developer - `Source/PinWright/Private/Handlers/GameplayTags/GameplayTagsHandler.cpp`: new `gameplay_tags.find_referencers` (required non-empty `tags[]`; per tag `{tag, registered, referencerCount, referencers[{packageName, objectName?}]}` in request order; `registryScanInProgress:true` while the initial scan runs; empty or non-string entries -> `INVALID_PARAMS`). `gameplay_tags.remove` on the whole-tag path now walks the tag plus every implicit parent the delete would take (mirror of `DeleteTagFromINIInternal`) and, on the first referenced name, returns error `TAG_IN_USE` with data `{tag, source, removed:false, reason:"referenced", blockingTag, referencerCount, referencers[]}` without calling the engine; non-removals carry `reason` `not_registered` / `implicit` / `not_in_source`; an engine `false` after the checks is now error `REMOVE_FAILED` instead of success. On 5.3 the pre-check is stricter than the engine (blocks on a referenced implicit parent too). New `ERR_TAG_IN_USE` in `Handlers/ErrorCodes.h`; the file's five hand-spelled codes converted to `ErrorCodes::` constants (registry adoption rule). Tests in `Tests/Gameplay/TestGameplayTagsHandlers.cpp`: `PinWright.gameplay_tags.FindReferencersListsSavedFixture` (saved `/Game/PinWrightTests` Blueprint with an `FGameplayTag` member is listed; unreferenced sibling reads 0; unregistered name reported `registered:false`; empty `tags` refused) and `PinWright.gameplay_tags.RemoveReferencedTagReportsReferencers` (`TAG_IN_USE` with the fixture package, tag still explicit afterwards, unreferenced sibling removed as control, `implicit` and `not_registered` reasons). Wiki: `docs/wiki-src/gameplay_tags.md` (prelude, `### gameplay_tags.remove`, `### gameplay_tags.find_referencers`); `docs/error-code-catalog.md` hand-annotated `TAG_IN_USE` row. Handler and test file compile-checked with `-SingleFile` on 5.8; other engines not compiled, suite not run.
