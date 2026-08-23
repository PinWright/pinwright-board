---
id: B-references-rejects-short-package-path
title: "asset.references / asset.dependencies return ASSET_NOT_FOUND for a short package path that asset.exists accepts — and whether it fails depends on whether the package happens to be loaded"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset, references, dependencies, asset-path, package-path, object-path, ASSET_NOT_FOUND, cross-verb-inconsistency, load-state-dependent]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# One string, accepted by the probe verb and refused by the two that use it

`asset.references` (`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:2915`,
registered `:2899`) and `asset.dependencies` (`:2976`, registered `:2960`) both do:

```cpp
FAssetData AssetData = AssetRegistry.GetAssetByObjectPath(FSoftObjectPath(AssetPath));
if (!AssetData.IsValid()) { Ctx.SendError(TEXT("ASSET_NOT_FOUND"), ...); return true; }
```

The caller string goes in raw — no `AssetPathParamUtils`, no `FPackageName::ObjectPathToPackageName`,
no inferred `.ObjectName`.

`asset.exists` (`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:721`, registered
`:713`) routes through `UEditorAssetLibrary::DoesAssetExist` →
`EditorAssetSubsystem.cpp:508-533` → `EditorScriptingHelpers.cpp:57-94`, which **infers the object
name from the package short name** at `:93` (`ObjectName = FPackageName::GetShortName(TextPath);`)
when the string carries no `.`. That single line is the whole divergence.

## Why it fails, and why only sometimes

`TopLevelAssetPath.cpp:145-152` — a dotless path sets `PackageName` and leaves `AssetName` empty
("Reference to a package itself. Iffy, but supported for legacy usage of FSoftObjectPath").
`AssetRegistry.cpp:2790-2817` then resolves in two steps: `FindObject<UObject>(nullptr,
"/Game/Foo/SM_Bar")` first — which for a package-only path returns the **UPackage if it is already
loaded**, yielding a valid `FAssetData` with the right `PackageName` — and otherwise falls to
`State.GetAssetByObjectPath`, keyed `Package.Asset`, which misses.

So the same call succeeds on a loaded asset and fails on an unloaded one. The reported "on some
assets" is really "depending on editor load state at the moment of the call", which is worse: it is
not reproducible from the path alone and will pass a test that touched the asset first.

## Correction: this is a visible error, not a silent false negative

The report framed it as a delete-path hazard — a caller checking referencers before deleting reads
the result as "nothing references it". That does not hold for these two verbs.
`Ctx.SendError(ASSET_NOT_FOUND)` produces `isError: true` with the code in the body
(`McpRequestCore::WrapToolResult`, `:826-860`). The caller gets a refusal it must handle, not an
empty referencer list. The severity here is cross-verb inconsistency, not silent data loss.

**The silent version is real, on a different verb.**
`Source/PinWright/Private/Handlers/Asset/AssetQueryHandler.cpp:16-72` (`asset.get_dependencies`) now
normalizes object-path → package name at `:27-30` but performs **no existence check at all**, so any
unresolvable path returns `{"dependencies": []}` with `isError: false`. That is the delete-path
hazard as described, and it is the surviving half of `B-get-dependencies-object-path-empty`
(IN-REVIEW) — whose object-path normalization has landed while its silent-empty half has not.

**Fix:** route both verbs' `assetPath` through the same normalization `asset.exists` gets, so a short
package path resolves identically across the three verbs regardless of load state. The
load-state-dependent success is the part to remove first — a verb that works only when the asset
happens to be in memory is worse than one that consistently refuses.

## Related

- `B-get-dependencies-object-path-empty` (IN-REVIEW) — the mirror defect on `asset.get_dependencies`
  / `asset.get_dependencies_classified`; normalization landed at `AssetQueryHandler.cpp:27-30`, the
  silent-empty-on-nonexistent half has not.
- `E-asset-dependencies-references-inverted` (IN-REVIEW) — direction naming, satisfied by the alias
  fields at `UtilityPropertyHandler.cpp:2946-2951` / `:3007-3012`. Different axis.
- `E-asset-path-vs-assetpath-list-drift`, `B-asset-list-short-class-ensure` — adjacent path-vocabulary
  tickets.

## History
- `#1-short-path-refused-and-load-state-dependent` `OPEN` reporter — `asset.references` (`UtilityPropertyHandler.cpp:2915`) and `asset.dependencies` (`:2976`) hand the caller's raw `assetPath` to `AssetRegistry.GetAssetByObjectPath(FSoftObjectPath(...))`, while the sibling probe `asset.exists` (`AssetManageHandler.cpp:721`) routes through `UEditorAssetLibrary::DoesAssetExist`, whose helper appends the inferred object name to a dotless path (`EditorScriptingHelpers.cpp:93`). A short package path therefore builds an `FSoftObjectPath` with an empty `AssetName` (`TopLevelAssetPath.cpp:145-152`) matching no registry key, and the two verbs return `ASSET_NOT_FOUND` for the exact string `asset.exists` accepts; the full `Pkg.Object` form works. The failure is **load-state dependent**, not per-asset: `GetAssetByObjectPath`'s first branch (`AssetRegistry.cpp:2796`) does a `FindObject` on the raw string, which resolves to the `UPackage` when that package is already in memory, so the same call succeeds on loaded assets and fails on unloaded ones — not reproducible from the path alone, and it will pass any test that touched the asset first. Correction to the report: this is a **visible** typed error, not a silent false negative — `Ctx.SendError(ASSET_NOT_FOUND)` yields `isError:true` with the code in the body (`McpRequestCore::WrapToolResult:826-860`), so a caller checking referencers before a delete gets a refusal it must handle, and the "reads as nothing references it" hazard does not apply here. That hazard is real on a different verb: `asset.get_dependencies` (`AssetQueryHandler.cpp:16-72`) normalizes the path at `:27-30` but performs no existence check, returning `{"dependencies":[]}` with `isError:false` for any unresolvable path — the surviving half of `B-get-dependencies-object-path-empty` (IN-REVIEW). Fix: route both verbs through the same normalization `asset.exists` gets; remove the load-state-dependent success first.

- `#2-already-landed-verified-at-head` `IN-REVIEW` developer — **Already fixed at HEAD; verified rather than re-implemented, and no new code was written.** Commit `0064cc61` routed all four verbs through the shared `ResolveAssetPathToPackage` (`Utils/AssetUtils.cpp:130`, contract in `Utils/AssetUtils.h:50-96`): `asset.references` (`Handlers/Utility/UtilityPropertyHandler.cpp:2926`), `asset.dependencies` (`:2989`), `asset.get_dependencies` and `asset.get_dependencies_classified` (`Handlers/Asset/AssetQueryHandler.cpp:40`). The load-state dependence is gone at the root: the resolver queries `GetAssetsByPackageName` instead of `GetAssetByObjectPath(FSoftObjectPath(...))`, so the `FindObject` fast path that made a dotless string succeed only for an already-loaded package is no longer on the path. The silent half named in this ticket is closed too — `asset.get_dependencies` now emits `ASSET_NOT_FOUND` for an unresolvable path instead of `{"dependencies":[]}` with `isError:false`. **`PathUtils::NormalizeToObjectPath` was checked as instructed and its subobject bug is fixed**: `Utils/PathUtils.cpp:172-190` splits a `:` suffix off BEFORE `IsValidAssetPath` (which rejects every `:` in order to turn away a Windows drive letter) and re-attaches it on both the object-path and package-path branches, so `/Game/A/BP_X.BP_X:Component` passes through unchanged; `Tests/Core/TestPathUtils.cpp:252-255` asserts exactly that, and `C:/Game/A/SM_X` is still rejected because splitting it leaves `C`, which has no leading slash. It is deliberately NOT what these four verbs adopted: their question is per-PACKAGE, and `ResolveAssetPathToPackage` additionally accepts `/Game/Foo/Bar.Baz` (an asset name differing from the package leaf) and a package holding several assets, which appending `.<leaf>` cannot handle. Both helpers are correct for their own job. Existing failure-direction coverage in `Tests/Assets/TestAssetPathResolution.cpp`: `PinWright.asset.references.ShortPackagePathFormResolves` deliberately searches for an UNLOADED /Game asset — the only state the defect was observable in — calibrates against `asset.exists` accepting the same string, and requires the short and long forms to agree; plus four `…UnresolvablePathIsError` tests, one per verb. Nothing further to implement; passing to a tester to confirm on a live editor.
