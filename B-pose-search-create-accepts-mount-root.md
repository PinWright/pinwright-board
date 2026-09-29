---
id: B-pose-search-create-accepts-mount-root
title: "pose_search.create_schema/create_database accept a bare mount root as assetPath and log a DoesPackageExist engine Error"
status: IN-REVIEW
severity: Low
category: bug
tags: [pose_search, paths, log-noise, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:27:02Z
---

# `assetPath: "/Game"` passes the path guard

`BuildCreatePaths` (`Source/PinWrightPoseSearch/Private/Handlers/PoseSearch/PoseSearchHandler.cpp`) validates with
`NormalizeAssetPath`, `SanitizeProjectRelativePath` and `FPackageName::IsValidLongPackageName`, all of which accept `/Game`. It
then composes the object path `/Game.Game`, and the ALREADY_EXISTS `LoadObject` on it logs
`LogPackageName: Error: DoesPackageExist: DoesPackageExist FAILED: '/Game/' is not a long packagename name.` (once or twice
depending on prior state) before the companion-asset lookup refuses the call with SKELETON_NOT_FOUND / SCHEMA_NOT_FOUND, an error
that names the wrong argument.

Observed in `PinWright.pose_search.CreateDatabaseMalformedAssetPathIsRefused` (2 lines) and `CreateSchemaMalformedAssetPathIsRefused`
(1 line), `Saved/Logs/pw_gapwave_full_offscreen2.log:44735-44736,44777`.

**Fix:** refuse a package path with no folder below its mount point (`FPackageName::GetLongPackagePath` empty) as INVALID_PATH, quoting the path.

## History
- `#1-mount-root-load-error` `OPEN` reporter — Found while triaging tests newly exposed by the per-test `bSuppressLogErrors` reset: the bare-mount-root case logged a DoesPackageExist engine Error in both create verbs.
- `#2-content-root-check` `IN-REVIEW` developer — `BuildCreatePaths` now answers INVALID_PATH ("names a content root, not an asset") when `GetLongPackagePath(OutPackagePath)` is empty, before any load. `TestPoseSearchCreateAssetPathSafety.cpp` drives `/Game` through `ExpectAssetPathRefused` (INVALID_PATH, message quotes the path) in both tests; the now-unused `ExpectAssetPathRefusedDownstream` helper is removed. Compile-checked (`-SingleFile`) only; needs a suite run.
