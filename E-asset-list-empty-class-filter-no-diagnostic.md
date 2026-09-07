---
id: E-asset-list-empty-class-filter-no-diagnostic
title: "asset.list's empty-result diagnostics are suppressed whenever the folder has subfolders, so a class filter that matches nothing returns zero rows and zero explanation"
status: OPEN
severity: Low
category: ergonomic
tags: [asset, asset-list, class-filter, diagnostics, empty-result, discoverability]
encounters: 1
costly: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The explanation only appears when there is nothing at all to report

`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:1031` gates the empty-result
diagnostics block on `TotalCount == 0 && FoldersJson.Num() == 0`. A class filter that matches no
asset inside a folder that **has subfolders** therefore produces zero rows, a non-empty `folders`
array, and no explanation of why the filter matched nothing.

That is the common case for a scoped query: `asset.list` on `/Game/Characters` with
`filter.class: "SkeletalMesh"` where the meshes live one level down returns an empty asset list next
to a populated folder list, and the caller cannot distinguish "no such class here" from "filter did
not apply" from "wrong case".

The third of those is a live defect: `:887` compares with `FString::Equals` and no `ESearchCase`, so
the post-filter is case-**sensitive** and a mis-cased `filter.class` silently returns nothing — filed
as `B-asset-list-class-filter-case-divergence` (OPEN, Medium). This ticket is the reason that defect
is hard to recognise from the response: the one mechanism that would have named it is switched off
in exactly the situation where a caller is most likely to hit it.

**Found while verifying a report** that `asset.list` with `filter.class: "ObjectRedirector"` returned
unrelated assets. That report did not reproduce — the class resolves correctly through `ResolveUClass`
(`:866`, `/Script/CoreUObject` is in the package list at `Utils/ClassUtils.cpp:151-167`), the path is
added to `FARFilter::ClassPaths` at `:871`, the post-filter at `:880-901` drops every row of another
class, and redirectors are not skipped by the registry (`AssetRegistryInterface.cpp:95-145` skips only
`BlueprintGeneratedClass` and `Blueprint`). The plugin's own `fixup_redirectors` uses the identical
filter shape (`AssetWorkflowHandler.cpp:68`, `:526`) and works. The likeliest source of the original
report is a class key passed at top level rather than inside `filter` (`:800-806` reads it only from
`filter`), which returned the whole folder unfiltered until `4fe1c420` turned it into a typed
rejection.

**Fix:** run the diagnostics block whenever `TotalCount == 0`, regardless of `FoldersJson`, and have
it name the resolved class path it filtered on plus the registry spelling it compared against. That
single line turns both the case divergence and a genuinely empty folder into self-describing
responses.

## Related

- `B-asset-list-class-filter-case-divergence` (OPEN, Medium) — the case-sensitivity defect this
  suppression hides.
- `B-asset-list-short-class-ensure` (DONE) — the short-name `FTopLevelAssetPath` ensure, fixed by the
  `ResolveUClass` routing at `:866`.
- `E-asset-list-path-ignored`, `E-asset-list-no-projection-spills` — adjacent `asset.list` ergonomics.

## History
- `#1-diagnostics-off-when-folders-present` `OPEN` reporter — `asset.list`'s empty-result diagnostics block is gated on `TotalCount == 0 && FoldersJson.Num() == 0` (`AssetManageHandler.cpp:1031`), so a class filter matching nothing inside a folder that has subfolders returns zero rows, a populated `folders` array and no explanation — leaving "no such class here", "filter did not apply" and "wrong case" indistinguishable. The third is real: the post-filter at `:887` uses `FString::Equals` with no `ESearchCase`, so a mis-cased `filter.class` silently returns nothing (`B-asset-list-class-filter-case-divergence`, OPEN, Medium), and this suppression is why that defect is hard to recognise from the response. Found while verifying a report that `filter.class: "ObjectRedirector"` returns unrelated assets; that did not reproduce — `ResolveUClass` (`:866`) finds it in `/Script/CoreUObject` (`Utils/ClassUtils.cpp:151-167`), it is added to `FARFilter::ClassPaths` (`:871`), the post-filter (`:880-901`) drops every row of another class, redirectors are not skipped by the registry (`AssetRegistryInterface.cpp:95-145` skips only `BlueprintGeneratedClass` and `Blueprint`), and the plugin's own `fixup_redirectors` uses the identical filter shape (`AssetWorkflowHandler.cpp:68`, `:526`). The likeliest source of that report is a class key passed at top level instead of inside `filter` (`:800-806` reads it only from `filter`), which returned the folder unfiltered until `4fe1c420` made it a typed rejection. Fix: run the diagnostics whenever `TotalCount == 0` regardless of `FoldersJson`, and name the resolved class path and the registry spelling compared against.
