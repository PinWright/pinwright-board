---
id: B-open-level-engine-mount-mangled
title: "editor.open_level mangles /Engine/ (and any non-/Game/ mount) into Content/e/... via a hardcoded RightChop(6)"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, open_level, level, path-resolution, mount-point, engine-content, file-not-found]
---

# `editor.open_level` corrupts `/Engine/`-mounted level paths

`editor.open_level` cannot open any level outside the `/Game/` mount. Given a
real, on-disk engine map path like `/Engine/Maps/Templates/OpenWorld`, the
handler silently rewrites it to the nonsensical filesystem path
`<ProjectContentDir>/e/Maps/Templates/OpenWorld.umap` and returns a false
`[FILE_NOT_FOUND]`. The path it claims is missing was never a real candidate —
the resolver mangled it.

## Root cause

`EditorCommandHandler.cpp:471-472` (the `editor.open_level` handler):

```cpp
FString MapPath = LevelPath + TEXT(".umap");
FString FullMapPath = FPaths::ProjectContentDir() + MapPath.RightChop(6);
```

The `RightChop(6)` blindly strips the first **6 characters**, hardcoding the
assumption that the package always starts with the 6-char prefix `/Game/`. That
holds for `/Game/Maps/MyLevel` (chop `/Game/` → `Maps/MyLevel.umap` under the
project content dir) but is wrong for every other mount:

- Earlier in the handler, `if (!IsValidMountPoint(LevelPath)) LevelPath =
  "/Game/" + LevelPath;` — and `IsValidMountPoint` accepts `/Engine`, `/Script`,
  and plugin roots (see `PathUtils.h:7-15`). So a `/Engine/...` path is left
  untouched (no `/Game/` is prepended), then falls into the unconditional
  `RightChop(6)`.
- For `/Engine/Maps/Templates/OpenWorld.umap`, `RightChop(6)` strips `/Engin`
  (6 chars) and leaves `e/Maps/Templates/OpenWorld.umap`, which is then
  appended to `FPaths::ProjectContentDir()` → `.../Content/e/Maps/...`. That
  exactly produces the observed garbage path.

This also breaks `/Game`-without-trailing-slash and any plugin mount (e.g.
`/MyPlugin/...` would chop `/MyPlu`).

The correct resolution is mount-point aware:
`FPackageName::LongPackageNameToFilename(LevelPath, TEXT(".umap"))` (or
`TryConvertLongPackageNameToFilename`), which maps `/Engine/` →
`<Engine>/Content/`, `/Game/` → `<Project>/Content/`, and plugin mounts to their
real content dirs — instead of string-chopping a fixed prefix length and forcing
the project content dir.

## The sibling proves the fix is achievable

`level.load` (the synchronous sibling, same `{levelPath}` param) resolves the
identical path correctly. Replayed live this pass:

`level.load {levelPath:"/Engine/Maps/Templates/OpenWorld"}` →
`{"alreadyLoaded":true,"levelPath":"/Engine/Maps/Templates/OpenWorld","verifiedPath":"/Engine/Maps/Templates/OpenWorld","existsAfter":true}`

So `/Engine/` content is loadable; only `editor.open_level`'s bespoke path math
is broken. (`level.load` and `editor.open_level` are documented as
sync/async variants of the same operation — they should resolve paths
identically.)

## Impact

Engine template maps (`/Engine/Maps/Templates/OpenWorld`, the canonical
open-world starting point) and any plugin-mounted level are unreachable through
`editor.open_level`. An agent that reaches for the documented async opener for
the obvious engine path gets a `FILE_NOT_FOUND` that names a path the user never
supplied, sending them down a path-form-guessing rabbit hole. The wiki page only
mentions `/Game/`, but the handler accepts other mounts far enough to produce a
corrupted result rather than a clean "only /Game/ supported" rejection — the
silent corruption is the bug.

**Workaround:** Use `level.load` instead for non-`/Game/` mounts.

**Fix:** Replace the `FPaths::ProjectContentDir() + MapPath.RightChop(6)`
construction at `EditorCommandHandler.cpp:471-472` with
`FPackageName::LongPackageNameToFilename(LevelPath, TEXT(".umap"))` (guard the
boolean `TryConvert*` variant and emit a clean error if conversion fails). This
also fixes plugin mounts and `/Game` without a trailing slash. Note `MapPath`
is still passed verbatim to `FEditorFileUtils::LoadMap` in the deferred job
lambda (line 502) — `LoadMap` takes a package/filename and would itself choke on
the mangled value, so the existence pre-check is the only thing that surfaces it
today; fixing the resolution fixes both.

## Repro (verbatim, replayed live)

1. `editor.open_level {levelPath:"/Engine/Maps/Templates/OpenWorld"}` →
   `[FILE_NOT_FOUND] Level file not found: X:/src/unreal/EAContentExamples57-fuzz1/Content/e/Maps/Templates/OpenWorld.umap`
2. `editor.open_level {levelPath:"/Engine/Maps/Templates/Template_Default"}` →
   `[FILE_NOT_FOUND] Level file not found: X:/src/unreal/EAContentExamples57-fuzz1/Content/e/Maps/Templates/Template_Default.umap`
   (second `/Engine/` path → identical `Content/e/...` mangling; deterministic)
3. Contrast — `level.load {levelPath:"/Engine/Maps/Templates/OpenWorld"}` →
   `{"alreadyLoaded":true,...,"verifiedPath":"/Engine/Maps/Templates/OpenWorld","existsAfter":true}` (resolves correctly)
4. The mangled path is fictional; the real file exists at
   `C:/UE_5.7/Engine/Content/Maps/Templates/OpenWorld.umap` (verified on disk).

## History
- `#1-initial-repro` `OPEN` reporter — Found via a streaming/World-Partition
  task that asked to open `/Engine/Maps/Templates/OpenWorld`. Replay-confirmed
  deterministically: two distinct `/Engine/` paths both mangle the prefix to
  `Content/e/...` and 404. Root cause is the hardcoded `MapPath.RightChop(6)` +
  forced `FPaths::ProjectContentDir()` at `EditorCommandHandler.cpp:471-472`,
  which assumes the 6-char `/Game/` prefix; `IsValidMountPoint` (`PathUtils.h:7-15`)
  lets `/Engine`/`/Script`/plugin mounts through unprepended so they hit the bad
  chop. Sibling `level.load` resolves the same `/Engine/` path correctly
  (`verifiedPath` echoed, `existsAfter:true`), proving the fix target. Distinct
  from the DONE `B-editor-open-level-no-completion-signal` (completion-delegate,
  not path resolution) and from `E-level-load-file-not-found-vs-in-memory-orphan`
  (a correctly-resolved-but-unsaved-in-memory world; here the path itself is
  corrupted before any disk check). Proposed fix:
  `FPackageName::LongPackageNameToFilename(LevelPath, TEXT(".umap"))`.
- `#2-mount-aware-resolve` `IN-REVIEW` developer — Replaced the broken
  `MapPath.RightChop(6)` + `FPaths::ProjectContentDir()` prefix-chop in the
  `editor.open_level` handler with a mount-aware resolver. Added
  `ResolveLevelPackageToMapFilename(LevelPackageName, OutMapFilename)` to
  `Utils/AssetUtils.{h,cpp}` — it strips an optional trailing `.umap` and calls
  `FPackageName::TryConvertLongPackageNameToFilename(..., GetMapPackageExtension())`,
  the same resolver the sync sibling `level.load` uses, so `/Game`→project
  content, `/Engine`→engine content, and plugin roots→their real content dirs.
  The handler (`Handlers/Editor/EditorCommandHandler.cpp`, the `editor.open_level`
  body) now calls the helper, cleanly rejects an unregistered mount with
  `FILE_NOT_FOUND` ("no registered mount point") instead of corrupting it, runs
  the existence pre-check on the real resolved path, and passes the mount-resolved
  on-disk filename to `FEditorFileUtils::LoadMap` (works for every mount, not just
  `/Game`). Regression test added to
  `Private/Tests/Core/TestLevelSaveLoadUtils.cpp`:
  `...resolve_level_package_to_map_filename.EngineMount` asserts
  `/Engine/Maps/Templates/OpenWorld` (with and without `.umap`) resolves under the
  engine content dir, ends in `.umap`, keeps the `OpenWorld` name, and contains no
  `/Content/e/` mangling fragment; `...GameMount` asserts `/Game/Maps/MyLevel`
  still lands under project content and that an unregistered mount returns false
  rather than a mangled path. Both fail if the helper is reverted to the
  `RightChop(6)` chop. Not compiled/tested here (later phase).
</content>
</invoke>
