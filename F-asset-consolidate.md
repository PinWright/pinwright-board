---
id: F-asset-consolidate
title: "No asset.consolidate: merging duplicate assets (repoint referencers onto one survivor, delete the rest) needs python.execute + EditorAssetLibrary.consolidate_assets and leaves unreported dirty packages"
status: OPEN
severity: Medium
category: feature
tags: [asset, consolidate, replace-references, dedup, referencers, redirectors, dirty-packages, python-fallback]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
rice: [1, 2, 1, 3]
priority: 6
---

# No verb merges duplicate assets into one survivor

The `asset.*` namespace has `rename`, `move` and `bulk_rename` (`Handlers/Asset/AssetManageHandler.cpp:907`, `:1018`; `AssetWorkflowHandler.cpp:613`), `fixup_redirectors` (`AssetWorkflowHandler.cpp:225`) and `delete` / `bulk_delete` (`AssetManageHandler.cpp:1158`, `AssetWorkflowHandler.cpp:907`). None of them repoints the referencers of asset A onto an existing asset B. `asset.rename` goes through `UEditorAssetLibrary::RenameAsset` (`AssetManageHandler.cpp:960`), which cannot land on an occupied path, and `asset.delete force:true` repoints referencers to **null**, not to a replacement (see `Utils/AssetDeletePolicy.h:12-35`). The only `ObjectTools::ForceReplaceReferences` calls in the plugin are internal to animation import (`AnimationHandler.cpp:1416`, `:1421`, `:1470`), and no handler calls `ConsolidateObjects`.

Evidence: to deduplicate a vendor pack (merge 8 duplicates, move 11), an agent had to run `python.execute` calling `EditorAssetLibrary.consolidate_assets` and `AssetTools.rename_assets`. Afterwards `editor.list_dirty_packages` showed 4 unexpectedly dirty packages, which the agent discarded with `editor.quit {discard:true}` without knowing whether they held the consolidation's own edits. It then ran `asset.fixup_redirectors` by hand.

**Workaround:** use `python.execute` with `unreal.EditorAssetLibrary.consolidate_assets(target, [sources])`, then `editor.list_dirty_packages` and `asset.save` for each referencer, then `asset.fixup_redirectors`. Nothing reports which packages the consolidation dirtied, so you cannot tell its collateral from other dirty packages.

**Fix:** add `asset.consolidate {target, sources:[...], save?:true, fixupRedirectors?:true}` that:
- refuses on a class mismatch (with each class named) and when a source is the target;
- runs `ObjectTools::ConsolidateObjects`, keeping the `FConsolidationResults` / dirtied-package set that the Python wrapper throws away;
- saves the repointed referencers;
- reports `referencersRepointed[]`, `sourcesDeleted[]`, `redirectorsLeft[]` and `dirtyPackagesLeft[]`.

Design risk: `ConsolidateObjects` is built on the same global `ForceReplaceReferences` walk as `B-force-delete-nulls-referencers` and `B-asset-import-overwrite-commit-kills-editor`. Before running it, check that no asset editor is open on any referencer, and scope `ObjectsToReplaceWithin` to the referencer set if the engine API permits it.

## History
- `#1-python-dedup-dirty-leftovers` `OPEN` reporter — A vendor-pack dedup needed a `python.execute` script with `consolidate_assets` and `rename_assets` (8 merged, 11 moved). It left 4 unexplained dirty packages, which were discarded through `editor.quit {discard:true}`, and `asset.fixup_redirectors` had to be run by hand. Source check: no consolidate or replace-references verb exists, and `rename`, `move` and `delete force:true` cannot merge an asset onto an existing one.
