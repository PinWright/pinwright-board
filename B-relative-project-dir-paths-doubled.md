---
id: B-relative-project-dir-paths-doubled
title: "File-path verbs double the project dir onto engine-produced paths when FPaths::ProjectDir() is relative, and mrq.create_job refuses every '{project_dir}' output directory"
status: IN-REVIEW
severity: High
category: bug
tags: [paths, linux, project-dir, image, level.export, model.compile, mrq, create_job]
encounters: 1
lastSeen: 2026-09-30T11:42:00Z
---

# Engine-produced relative paths get the project dir joined on twice

Where the project shares a filesystem root with the engine, `FPaths::ProjectDir()` is not
absolute: it is relative to the process BaseDir (`../../../../<path-to-project>/`), and so is
every `Project*Dir()` built on it (`ProjectSavedDir()`, `ProjectIntermediateDir()`,
`ProjectContentDir()`), plus the paths the engine prints in its own log lines.

Handlers that accept a filesystem path treated any relative path as project-relative and joined
it onto `ProjectDir()`. An engine-produced path therefore came out as
`../../../../proj/../../../../proj/Saved/...`, which collapses past the filesystem root (for
example to `/src/...`) and fails with `FILE_NOT_FOUND`, `MODEL_FILE_NOT_FOUND`, a failed
exporter open or `EACCES` on `create dir('/src/')`. Sites: `ImageOps.cpp` `ResolveInputPath` /
`ResolveOutputDir` (image.annotate / compare / tile), `LevelHandler.cpp` level.export,
`ModelCompileHandler.cpp` / `SkeletonCompileHandler.cpp` / `AnimCompileHandler.cpp` source
resolution, `AssetWorkflowHandler.cpp` thumbnail output, `SequencerFbxHandler.cpp`,
`WikiDiskGenerator.cpp` `WikiOutputDirectory`, and `AssetDumpWriter.cpp` `ResolveDumpRoot`.

`mrq.create_job` is the user-facing half: `MRQHandler.cpp` `ValidateMRQOutputDirectory` expanded
`{project_dir}` with the relative `ProjectDir()` and then refused the result for containing
`..`, so every job whose preset keeps MRQ's default `{project_dir}/Saved/...` output directory
was refused with `INVALID_PATH` on such a host.

Invisible wherever `ProjectDir()` is absolute (engine and project on different Windows drives),
because no relative path can then begin with it.

Related: `B-dump-root-double-prefixes-project-dir` fixed the dump-root instance by changing the
test callers to pass absolute roots; the resolver itself still doubled an engine-produced root.

**Evidence:** host-project run `Saved/PinWright/test-runs/693f8d20ebbd4e59979c3e44d76cfdbd/automation.log`,
23 failures: 7 `image.*`, 2 `level.export.*`, 8 `Model.Handlers.*`, 6 `mrq.create_job.*`
(host-project path; a fresh clone will not have it, re-measure).

## History
- `#1-initial-repro` `OPEN` reporter — 23 suite failures on a Linux host whose `FPaths::ProjectDir()` is `../../../../src/unreal/unreal-fpv/`: image / level.export / model.compile fixtures pass `ProjectSavedDir()` / `ProjectIntermediateDir()` / `ProjectContentDir()`-derived paths and the handlers join them onto `ProjectDir()` again; mrq.create_job refuses the `{project_dir}` expansion as traversal.
- `#2-shared-resolver` `IN-REVIEW` developer — Added `ResolveProjectFilePath` to `Source/PinWright/Private/Utils/PathUtils.{h,cpp}`: a relative path is still project-relative (joined onto the ABSOLUTE project dir), except one that already begins with `FPaths::ProjectDir()`, which is resolved against BaseDir as the engine means it; absolute paths pass through. Routed every bypassing site through it: `ImageOps.cpp` (`ResolveInputPath`, `ResolveOutputDir`), `LevelHandler.cpp` (level.export), `ModelCompileHandler.cpp`, `SkeletonCompileHandler.cpp`, `AnimCompileHandler.cpp`, `AssetWorkflowHandler.cpp`, `SequencerFbxHandler.cpp`, `WikiDiskGenerator.cpp`, `AssetDumpWriter.cpp`. `MRQHandler.cpp` now expands `{project_dir}` with `ConvertRelativePathToFull(ProjectDir())` (`docs/wiki-src/mrq.md` says so). No test fixture changed: the fixtures pass engine-produced paths, which callers legitimately do. New regression test `PinWright.core.path.resolve_project_file_path.EngineProducedPathResolvesOnce` (`Tests/Core/TestPathUtils.cpp`); the 23 previously failing tests are the handler-level coverage. Compile-checked single-file; needs a suite run on the relative-ProjectDir host for DONE.
