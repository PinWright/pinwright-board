---
id: E-asset-dump-distinguish-file-missing-from-load-failed
title: "asset.dump conflates 'source .uasset absent on disk' with 'asset present but failed to load' under one ASSET_LOAD_FAILED code"
status: DONE
severity: Low
category: ergonomic
tags: [asset-dump, error-codes, skip-reasons, telemetry]
---

# asset.dump conflates 'source .uasset absent on disk' with 'asset present but failed to load' under one ASSET_LOAD_FAILED code

## Observed

`DumpSingleAsset` in `Handlers/Asset/AssetDumpHandler.cpp` (around line 1190–1200) is the sole emit site for `ASSET_LOAD_FAILED`. It calls `LoadObject<UObject>(nullptr, *NormalizedPath)`, and if that returns null it writes a skip stub with:

```
ErrorCode    = AssetDumpErrorCodes::AssetLoadFailed   // "ASSET_LOAD_FAILED"
ErrorMessage = FString::Printf(TEXT("Failed to load asset at path '%s'"), *NormalizedPath);
```

That branch fires for two qualitatively different conditions:

1. **The source `.uasset` file truly doesn't exist on disk** — typically AssetRegistry-enumerated stub entries left behind by package bakers (auto-generated MaterialInstanceConstants from material baking, content-pack residue, orphan re-import artifacts). `LoadObject` returns null because there's nothing to load.
2. **The file exists but cannot be loaded** — corrupt content, missing/renamed dependency, deprecated class, version-mismatched payload, etc. `LoadObject` returns null because the load itself failed.

A consumer reading skip stubs cannot tell these apart. Both look identical:

```json
{
  "skipped": true,
  "skipReason": "ASSET_LOAD_FAILED",
  "skipMessage": "Failed to load asset at path '/App/.../<asset_with_guid>'"
}
```

The two cases need different responses. Case 1 is "ignore, it was never going to load" — typically third-party / generated content, not actionable inside this project. Case 2 is "investigate" — real load failure worth chasing.

## Concrete examples (from prior `B-asset-dump-game-high-skip-ratio` investigation)

All four `/App` skips and 197/207 `/Game` skips in that ticket fall into **case 1** (file truly absent or orphan-stub residue):

- 3× `App/Meadow_Environment_Set2/Environment/Rocks/Meshes/M_SM_Cliff_*_MI_*_<32-hex-guid>` — auto-generated material baker products from a vendored Meadow pack
- 1× `App/Gates/Models/M_gate_02_gate_02_B_geo_gate_syntethic_<32-hex-guid>` — same shape on a Gate asset
- 197× `Game/UltraDynamicSky/.../UDS_*_2.uasset` — orphan duplicate copies from a botched marketplace re-import

The `B-asset-dump-game-high-skip-ratio` ticket already concluded these are content-side hygiene, not plugin defects — but it lumped them all under one code, which is exactly why distinguishing them at emit time is useful.

## Fix

Before calling `LoadObject`, resolve the `NormalizedPath` to a filesystem path and check existence with `FPaths::FileExists` (or equivalent — `FPackageName::DoesPackageExist` is the more UE-idiomatic call and already handles the `/Game/Foo` → `Content/Foo.uasset` mapping). If the package file does not exist, emit a distinct code:

```cpp
namespace AssetDumpErrorCodes
{
    inline constexpr const TCHAR* AssetFileMissing = TEXT("ASSET_FILE_MISSING");
    inline constexpr const TCHAR* AssetLoadFailed  = TEXT("ASSET_LOAD_FAILED");
    // ...
}
```

And in `DumpSingleAsset`:

```cpp
FString PackageFilename;
if (!FPackageName::DoesPackageExist(NormalizedPath, &PackageFilename))
{
    Result.ErrorCode    = AssetDumpErrorCodes::AssetFileMissing;
    Result.ErrorMessage = FString::Printf(
        TEXT("No package file on disk for asset path '%s'"), *NormalizedPath);
    return Result;
}

UObject* Asset = LoadObject<UObject>(nullptr, *NormalizedPath);
if (!Asset)
{
    Result.ErrorCode    = AssetDumpErrorCodes::AssetLoadFailed;
    Result.ErrorMessage = FString::Printf(
        TEXT("Failed to load asset at path '%s'"), *NormalizedPath);
    return Result;
}
```

The pre-check is also a tiny perf win — failing the existence test is much cheaper than spinning up the full LoadObject path only to bail.

## Why it matters

- Consumers triaging skip lists can filter `ASSET_FILE_MISSING` out as "not interesting" (orphan stubs, baked artifacts, content-pack residue) and focus on `ASSET_LOAD_FAILED` as the genuine "asset is broken" signal worth investigating.
- Aggregation telemetry (cf. `B-asset-dump-game-high-skip-ratio` `#1`'s proposed `skipCountsByReason`) becomes meaningful — `{ ASSET_FILE_MISSING: 207, ASSET_LOAD_FAILED: 0 }` tells a very different story than the current single-bucket count.
- It surfaces the real signal: a non-zero `ASSET_LOAD_FAILED` count post-split would indicate actual project rot worth chasing.

## Scope

Single emit site, single new error code constant in the central registry (`AssetDumpHandler.h` `AssetDumpErrorCodes` namespace). No changes to skip-stub schema beyond the reason string. Doc note in `wiki/asset.md` (or wherever skip reasons are catalogued) listing the new code.

## History
- `#1-initial-finding` `OPEN` reporter — `DumpSingleAsset` at `AssetDumpHandler.cpp:1193-1200` emits `ASSET_LOAD_FAILED` for both genuine load failures and assets whose source `.uasset` doesn't exist on disk. Skip-stub investigation against `B-asset-dump-game-high-skip-ratio` confirms all observed /App skips (3 GUID-suffixed Meadow MICs + 1 Gate asset) and most /Game skips (UDS `*_2.uasset` orphans) fall into the file-truly-absent case. Propose splitting into `ASSET_FILE_MISSING` (pre-check via `FPackageName::DoesPackageExist`) vs `ASSET_LOAD_FAILED` (file present, LoadObject returned null) so consumers can filter orphan/baker residue away from genuine load-failure investigations.
- `#2-file-missing-split` `IN-REVIEW` developer — Added `AssetFileMissing = TEXT("ASSET_FILE_MISSING")` to the `AssetDumpErrorCodes` namespace in `Handlers/Asset/AssetDumpHandler.h`. In `Handlers/Asset/AssetDumpHandler.cpp::DumpSingleAsset`, inserted a `FPackageName::DoesPackageExist` pre-check before `LoadObject`. The check converts the object path to a package name first via `FPackageName::ObjectPathToPackageName` so the `.<asset name>` suffix doesn't trip the package-name parser. Returns `ASSET_FILE_MISSING` with message "No package file on disk for asset path '<path>'" when the package isn't on disk; keeps `ASSET_LOAD_FAILED` for the file-present-but-LoadObject-null branch. Skip-stub propagation through `TickFolderDump` is unchanged — `skipReason` simply carries the new code as needed. `docs/wiki/asset.md` updated with the new error-code entry. Bundled into the schema-v6 bump.
- `#3-verify-fix` `DONE` tester — Editor offline, so verified at the source level (live MCP unreachable on :19880). Confirmed `AssetDumpHandler.h:64` defines `AssetFileMissing = TEXT("ASSET_FILE_MISSING")` in the central `AssetDumpErrorCodes` namespace. `AssetDumpHandler.cpp:1336-1344` derives the package name via `FPackageName::ObjectPathToPackageName` then checks `!InMemoryPackage && !FPackageName::DoesPackageExist(...)` before `LoadObject`, emitting `ASSET_FILE_MISSING` with the spec'd "No package file on disk for asset path '<path>'" message; the LoadObject-null branch at lines 1346-1353 still emits `ASSET_LOAD_FAILED`. The in-memory-package exemption is a sensible extension beyond spec (covers transient/unsaved editor packages without breaking orphan-stub detection). `docs/wiki/asset.md:176` documents the new code and changelog entry at line 333 records the split. `TestAssetDumpHandler.cpp` lines 365-374 and 2183-2200 accept `ASSET_FILE_MISSING` as the expected error code for synthetic absent paths. All five IN-REVIEW claims (constant, pre-check, two-branch behavior, wiki doc, schema-v6 bundling) are present in the diff.
- `#4-live-mcp-verify` `DONE` tester — Live MCP verification (prior `#3` was source-only, which protocol forbids). Called `asset.dump` with `assetPath="/Game/__DefinitelyDoesNotExist_McpVerifyTemp_EAssetDumpDistinguish/W_NoSuchAsset"` — response: `[ASSET_FILE_MISSING] No package file on disk for asset path '/Game/__DefinitelyDoesNotExist_McpVerifyTemp_EAssetDumpDistinguish/W_NoSuchAsset'`. Exact error code and message format from the IN-REVIEW spec. Also confirmed `docs/wiki/asset.md:175-177` documents the split with both `ASSET_FILE_MISSING` and `ASSET_LOAD_FAILED` as distinct codes, referencing the `FPackageName::DoesPackageExist` pre-check. Pre-check path proven end-to-end via live RPC.
