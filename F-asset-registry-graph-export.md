---
id: F-asset-registry-graph-export
title: "No whole-project asset dependency export, missing-dependency scan, or unreferenced-asset closure; asset.get_asset_graph is single-root and /Game-only"
status: OPEN
severity: Medium
category: feature
tags: [asset, dependencies, referencers, asset-registry, content-cleanup, reachability, missing-dependencies]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# No project-wide dependency graph, missing-dependency scan, or reachability closure

Every dependency verb takes one `assetPath` root: `asset.get_dependencies` (`AssetQueryHandler.cpp:18`), `asset.get_dependencies_classified` (`AssetMetadataHandler.cpp:327`), `asset.references` / `asset.dependencies` (`UtilityPropertyHandler.cpp:3461` / `:3532`), and `asset.get_asset_graph` (`AssetMetadataHandler.cpp:600`). `get_asset_graph` also calls `GetDependencies` with the default category (`:645`) and drops every edge not starting with `/Game` (`:651`), so plugin mounts such as `/App` vanish from the graph and hard/soft edges come back unlabeled. No verb lists packages whose dependencies point at a package that does not exist, and none computes the unreferenced set from a set of roots. The only project-wide referencer-style verb is `gameplay_tags.find_referencers`, which covers tags only.

Content cleanup needs all three answers at once. In one session four agents wrote their own Asset Registry Python to get them: `ar_check.py` (`scanned 1129 packages with missing deps 4`), `c2_ar_check.py` (`packages 1298 with missing deps 4`), `export_graph.py` (first run failed with `AttributeError: ... 'AssetManagerSettings'`), and `phaseb_referencers.py` over 654 packages.

**Workaround:** `python.execute` with `unreal.AssetRegistryHelpers.get_asset_registry()`: enumerate packages per mount with `get_assets_by_path(recursive=True)`, pull edges with `get_dependencies(pkg, AssetRegistryDependencyOptions(...))`, and flag an edge as missing when `get_assets_by_package_name(target)` returns nothing and the target is not a `/Script` package. Do the BFS closure in Python. Parse `PackageRedirects` from config first so redirect-shadowed paths are not reported as unreferenced and deleted.

**Fix:** Add `asset.dependency_graph {mounts, categories (hard|soft|searchable_name|manage), wait:false}`. It writes a JSONL file (one package per line with labeled edges) and returns the path plus counts, using the job pattern so a scan of the whole project cannot block the editor. On top of the same scan, add `asset.find_missing_dependencies {mounts}` (the referencing package, the missing target and the edge kind) and `asset.unreachable {roots, mounts}` (packages not reachable from the roots; roots default to the maps and primary assets that AssetManager cooks). Mark redirect-shadowed and redirector packages in the output instead of listing them as unreferenced. While in that code, drop the hardcoded `/Game` filter in `get_asset_graph` and replace it with a `mounts` parameter. Keep it game-agnostic: no project paths.

## History
- `#1-project-wide-graph-gap` `OPEN` reporter — Verified from source: all five dependency/referencer verbs take a single `assetPath` root, and `get_asset_graph` filters edges to `/Game` at `AssetMetadataHandler.cpp:651`. None detects missing targets or computes an unreferenced closure. There is no duplicate on the board: `E-asset-dependencies-references-inverted`, `B-get-dependencies-object-path-empty` and `F-gameplay-tag-referencers` cover per-asset naming, path normalization and tag referencers. Evidence: four ad-hoc Asset Registry scripts in one content-cleanup session (1129/1298/654-package scans, 4 packages with missing deps each, and an `AssetManagerSettings` AttributeError on first run).
