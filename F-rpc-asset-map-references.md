---
id: F-rpc-asset-map-references
title: "Add live RPC `asset.map_references` (TSoftObjectPtr<UWorld> walker)"
status: DONE
severity: Low
category: feature
tags: [asset, dump-parity, soft-references, world]
---

# Add live RPC `asset.map_references` (TSoftObjectPtr<UWorld> walker)

`PropertyUtils::BuildMapReferencesJson` walks an asset's `TSoftObjectPtr<UWorld>` UPROPERTYs (including arrays, sets, and maps) and emits the per-property world-reference list as `map_references.json`. The only call site is `AssetDumpHandler::BuildAssetDumpFiles` — there is no live RPC that returns the same data for a single asset on demand. Consumers who want to ask "which worlds does this asset's UPROPERTYs soft-reference, and via which property name?" must read the dump cache (or trigger a fresh `asset.dump` and read the sidecar).

`asset.get_dependencies` does not cover the gap: it returns asset-registry hard package dependencies for the entire package and is not filtered to `UWorld` targets, nor does it report which UPROPERTY held the reference. `asset.dump` documents `map_references.json` as a dump-only sidecar in `Docs/wiki/asset.md`.

This violates the dump-non-exclusivity policy: every dump output should also be reachable through a live RPC.

**Fix:** Add `asset.map_references` handler that takes `assetPath`, loads the asset, and returns the same JSON shape `BuildMapReferencesJson` already produces: `{ schemaVersion: 1, references: [{ property, path, source }] }`. When the asset cannot be loaded or the class has no qualifying soft-UWorld refs, return `{ schemaVersion: 1, references: [] }`. Reuse `BuildMapReferencesJson` directly; no new traversal code needed. Update `docs/wiki/asset.md` to list the live RPC alongside the dump sidecar.

This is a niche surface — most consumers will be fine with the dump cache — but the implementation cost is tiny (one handler delegating to an existing, already-tested utility) and it closes a policy hole.

## History
- `#1-initial-repro` `OPEN` reporter — `PropertyUtils::BuildMapReferencesJson` is wired only into `AssetDumpHandler.cpp:453` (writes `map_references.json`). Greps for `map_references` / `world_references` / `IsSoftWorldProperty` find no other handler call site. `asset.get_dependencies` returns hard package deps with no UWorld filter and no source-property attribution, so it does not substitute. Documented gap in `Docs/wiki/asset.md` (sidecar listed under `asset.dump` only).
- `#2-reformulated-response-shape` `OPEN` developer — Reformulated the task body to match the existing utility contract: `BuildMapReferencesJson` emits `{ schemaVersion, references: [{ property, path, source }] }`, not `{ property, worldPath }`.
- `#3-added-live-rpc` `IN-REVIEW` developer — Added the live `asset.map_references` handler in `AssetQueryHandler.cpp`, documented the RPC and `map_references.json` schema in `docs/wiki/asset.md`, and added regression `FAssetMapReferencesHandler_TypedSoftWorldEmittedTest` for the package-backed fixture path/source response.
- `#4-verify-fix` `DONE` tester — Verified: `asset.map_references` on `/App/App/Experiences/CabinLake/DA_CabinLake` returned `{schemaVersion:1, references:[{property:"Map", path:"/Game/Maps/L_PDS_Maldives.L_PDS_Maldives", source:"soft-uworld"}]}`, byte-identical to the existing `map_references.json` sidecar. Empty-path check on `/Engine/EngineMaterials/DefaultMaterial` returned `{schemaVersion:1, references:[]}`. Wiki page `asset.map_references` is present and documents the schema.
