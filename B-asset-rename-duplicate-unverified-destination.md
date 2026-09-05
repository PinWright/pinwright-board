---
id: B-asset-rename-duplicate-unverified-destination
title: "asset.rename and single asset.duplicate report the requested destination when post-operation resolution fails"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, rename, duplicate, destination, verification, request-echo, false-success]
---

# `asset.rename` and single `asset.duplicate` do not prove their result exists

## What's wrong

After the engine returns true, single `asset.duplicate` immediately publishes the requested
`DestinationPath` (`AssetManageHandler.cpp:492-510`). It attempts `ResolveAsset` only to add
optional verification; a null result still sends success. `asset.rename` has the identical shape
at `:568-580`.

This is already solved in the sibling `asset.move` path at `:688-720`: it resolves the destination,
fails `VERIFICATION_FAILED` when no object is found, and reports `MovedAsset->GetPathName()`.
Rename/duplicate did not receive that fix. A provider or registry delay, path normalization, or
engine-side destination adjustment therefore produces a green response containing only the
caller's requested string—not an observed asset.

## What it should do

Apply `asset.move`'s postcondition to both verbs: resolve the destination after the engine call,
fail if it cannot be resolved, and return the resolved object's canonical path and verification.

## Workaround

Call `asset.validate` on the returned destination and treat a failure as operation failure.

## Related

- `B-asset-move-to-missing-folder-silently-renames` fixed the sibling move path only.
- `B-asset-verification-clobbers-handler-assetpath` concerns verification field ownership, not a
  missing postcondition.

## Fix

Verdict: TRUE. Both single-asset branches called `ResolveAsset` only as an optional enrichment and
sent success when that read-back was null. The handler now reconciles the engine result with a
loadable destination object at the requested package, verifies that a duplicate still has its
source, and verifies that a rename leaves no non-redirector source asset. Responses use the
canonical observed object path and report registry/disk state (`sourceObservedPath`,
`sourceExistsAfter`, `sourceExistsOnDisk`, `sourceIsRedirector`, `destinationObservedPath`,
`destinationExistsAfter`, `destinationExistsOnDisk`). Postcondition failures use the typed
`DUPLICATE_FAILED` / `RENAME_FAILED` codes and carry the same read-back object. Folder duplicate
batch code and its ordered per-item contract were intentionally left unchanged.

Files changed:

- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetManageHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetManageHandlerTestHooks.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\TestAssetRenameDuplicateVerification.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Docs\wiki-src\asset.md`

Behavioral tests: `PinWright.asset.duplicate.ReportsRegistryAndDiskReadback` and
`PinWright.asset.rename.ReportsDestinationAndSourceReadback`. Tests were not run per the task
boundary (no live editor/PIE, MCP, Build.bat, or UnrealEditor-Cmd). A separate tester should run
the two IDs in an editor with writable test content and inspect the failure-direction path where
the engine reports success but destination read-back is unavailable.

Verifier follow-up test IDs:
- `PinWright.asset.duplicate.RejectsMissingDestinationReadback`
- `PinWright.asset.rename.RejectsMissingDestinationReadback`

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the wrong-target/request-echo catalogs; no editor, build, test, or RPC run was performed.
- `#2-readback-guards-landed` `IN-REVIEW` developer — Implemented registry/load and package-file postconditions for single `asset.rename` / `asset.duplicate`, added harness tests, and documented the response fields; tests not run by task boundary.
