---
id: B-asset-batch-mutators-silently-narrow-request
title: "Asset batch mutators silently discard malformed or missing inputs and can report success for only the surviving subset"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, batch, partial-success, silent-drop, checkout, submit, rename, duplicate, lod]
---

# Asset batches change a narrower set than the caller requested

## What's wrong

Several scoped Asset handlers build a filtered work list and lose the rejected entries before
they decide success:

- `asset.source_control_checkout` drops non-string and unresolved paths at
  `AssetWorkflowHandler.cpp:175-203`, then reports the surviving package count as `checkedOut`.
- `asset.source_control_submit` repeats that filter at `:260-303` and can submit the valid subset
  with no failed/missing list.
- `asset.bulk_rename` drops non-strings, missing assets, and unchanged names at `:361-412`; an
  all-missing request returns success/`renamed:0` at `:414-421`, while a mixed request can rename
  the valid subset without naming omissions. Its `oldPath` is sampled from `Data.Asset` only
  after the rename (`:438-444`), so the field can contain the new object path as well.
- folder `asset.duplicate` counts successful children and declares the whole request successful
  when `duplicatedCount > 0` (`AssetManageHandler.cpp:420-470`).
- `asset.generate_lods` drops non-string and non-StaticMesh entries at
  `AssetWorkflowHandler.cpp:1152-1205`, then always sends success with only `processed`.

Concrete failure: a batch containing one valid asset and one typo can mutate/submit the valid
asset and return a success envelope that contains no record of the typo. Retrying or assuming the
requested set is uniform then produces divergent project state.

## What it should do

Validate every array element and target before mutation, or return one measured result per input
and make the envelope fail/partial when any requested entry was not completed. Counts must come
from post-operation readback, not the filtered request array.

## Workaround

Resolve and operate on one asset at a time, verifying each result independently.

## Fix

**Fix:** The root cause was five independent handlers deriving success from filtered work arrays,
so rejected inputs no longer existed when the response was built. `AssetBatchResult.h` now owns one
shared per-item outcome and measured aggregate contract. `AssetWorkflowHandler.cpp` uses it for
checkout, submit, bulk rename, and LOD generation; `AssetManageHandler.cpp` uses it for folder
duplication; `MeshRebuildRenderGuard.h` adds a slot-preserving overload without changing existing
callers; `ErrorCodes.h` supplies stable batch refusal codes; and `Docs/wiki-src/asset.md` documents
the wire behavior. Regression coverage is
`PinWright.asset.batch_mutators.PreserveItemsAndRefusePartial` in
`TestAssetBatchMutators.cpp`; it calls the production helper and dispatches all five registered
production handlers. The four array verbs use an engine StaticMesh plus malformed and missing rows
to assert synchronous default refusal, complete input order, and unchanged mesh state; folder
duplication uses a same-folder engine-content collision to exercise its production branch and
assert every discovered child is retained and unattempted. Registration and source ratchets remain
supplemental, including the mixed-child destination-conflict invariant.

Deliberately unchanged: single-asset `asset.duplicate` keeps its existing contract, runtime
failures do not claim transaction rollback, and the regression deliberately performs no live
source-control operation or asset mutation. This source-only change provides no build,
automation-run, editor-runtime, external MCP, or provider proof.

## History
- `#1-pattern-scan` `OPEN` reporter — Grouped because all five verbs share the same request-narrowing mechanism and fix contract. Source only; no editor, build, test, or RPC run.
- `#2-batch-outcome-contract` `IN-REVIEW` developer — "Changed AssetWorkflowHandler.cpp and AssetManageHandler.cpp to route all five asset mutators through a shared per-input outcome contract, refuse incomplete batches by default, and support explicit partial:true results."
