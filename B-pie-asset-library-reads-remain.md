---
id: B-pie-asset-library-reads-remain
title: "Non-Asset handlers still turn PIE-blocked EditorAssetLibrary reads into false missing-asset results"
status: IN-REVIEW
severity: High
category: bug
tags: [pie, assets, editor-asset-library, false-negative, actor, sequencer, delete]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Non-Asset handlers still misreport PIE-blocked asset reads

## What happens

Representative non-Asset paths still use `UEditorAssetLibrary` directly. Shape
spawning loads its built-in mesh and maps a null result to `ASSET_LOAD_FAILED`
(`Source/PinWright/Private/Handlers/Actor/SpawnShapeHandler.cpp:52-57`). Sequence
path resolution uses `DoesAssetExist` and `LoadAsset` before falling back
(`Handlers/Sequencer/SequenceHandler.cpp:192-198`). Asset-delete policy treats a
failed load as “nothing was deleted” and uses library listing/loading for directory
contents (`Source/PinWright/Private/Utils/AssetDeletePolicy.cpp:161-167,227-239`).

During PIE these editor-library reads can be blocked or return null for assets that
exist. The callers do not distinguish that state from a genuinely missing or
unloadable asset.

## Why it matters

Normal PIE workflows can receive trusted false negatives, including a delete result
that says nothing existed to delete. Severity is High under the silent-wrong-data
rule, with reach extending beyond the Asset namespace into unrelated handlers.

## What should happen

Move non-Asset callers to the PIE-safe registry/object resolver used by the repaired
Asset paths, distinguish PIE-blocked access from not-found, and add a source ratchet
plus representative PIE-aware tests for actor, Sequencer, and delete-policy reads.

## Workaround

Stop PIE before using affected handlers or deletion utilities.

## Related

- `B-asset-exists-duplicate-false-negative-in-pie` — wave-6 ticket that repaired
  Asset-namespace existence checks; its review exposed the remaining non-Asset uses.

## Fix

The remaining handlers and `AssetDeletePolicy` treated PIE-gated `UEditorAssetLibrary` read sentinels as missing or unloadable assets, while directory deletion still enumerated through the same gated API. All targeted handler and policy reads now use the shared registry/object resolver (`ResolveAsset`, `DoesAssetDirectoryExist`, and `GetAssetPathsUnderDirectory`); the allowlisted editor-library mutations are preceded by `PIE_ACTIVE` refusal gates in their RPC callers. Files changed: the 28 affected handler `.cpp` files, `Utils/AssetDeletePolicy.cpp`, `Utils/AssetUtils.cpp/.h`, and `Tests/Assets/TestAssetPieSafeResolution.cpp`; wiki contract notes were updated for animation, blueprint, editor, level, physics, save, and unattended pages. Tests: `PinWright.Assets.AssetResolution.AssetHandlersAvoidPieBlockedLibraryReads`, `PinWright.Assets.AssetResolution.RegistryAndLoadAvoidEditorAssetLibrary`, `PinWright.Assets.AssetResolution.PiePredicateMirrorsEngineGate`, `PinWright.Assets.AssetResolution.ActorGetComponentsReadsTransientAssetInPie`, `PinWright.Assets.AssetResolution.SequencerGetBindingsReadsTransientAssetInPie`, and `PinWright.Assets.AssetResolution.AssetDeletePolicyResolvesAndDeleteIsPieGated`. Deliberately unchanged: `Utils/ActorUtils.cpp` and `Utils/ClassUtils.cpp`, whose remaining library reads are outside this handler-tree ticket and are follow-up candidates.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification reopened the representative calls at `SpawnShapeHandler.cpp:52-57`, `SequenceHandler.cpp:192-198`, and `AssetDeletePolicy.cpp:161-167,227-239`; each still uses `UEditorAssetLibrary` without a distinct PIE-blocked outcome. No PIE session, deletion, build, test, editor, or MCP call was run. Severity High because multiple normal handlers can return a trusted missing/load-failed result for an existing asset.
- `#2-pie-safe-handler-reads` `IN-REVIEW` developer — Replaced the targeted PIE-blocked library reads with registry/object resolution, added `PIE_ACTIVE` gates for the remaining editor-library mutations, and extended the source ratchet across the handler tree plus `AssetDeletePolicy`. Static verification only; no build, editor, test, or MCP run.
- `#3-pie-safe-verifier-follow-up` `IN-REVIEW` developer — Corrected registry half-adoption (Blueprint 2 literal PIE gates, Editor 1 literal PIE gate, Physics 23 constant conversions), added dispatcher-backed simulated-PIE actor/Sequencer/delete-policy coverage, and renamed the textual gate-order ratchet. Static verification only; no build, editor, test, or MCP run.
