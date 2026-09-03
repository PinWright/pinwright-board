---
id: B-asset-rename-duplicate-unverified-destination
title: "asset.rename and single asset.duplicate report the requested destination when post-operation resolution fails"
status: OPEN
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

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the wrong-target/request-echo catalogs; no editor, build, test, or RPC run was performed.
