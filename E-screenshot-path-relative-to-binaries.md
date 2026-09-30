---
id: E-screenshot-path-relative-to-binaries
title: "editor.screenshot returns path relative to the engine binaries directory (../../../../src/...), which is unusable from the caller's working directory"
status: OPEN
severity: Low
category: ergonomic
tags: [editor, editor.screenshot, paths, linux, response-metadata]
encounters: 1
lastSeen: 2026-09-30T12:29:00Z
---

# Screenshot path is relative to Engine/Binaries/Linux

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2` (project and engine on the same filesystem root,
engine at `/sdb-disk/UE_5.8`), plugin `adb239fd`. Every `editor.screenshot` returned for example
`"path":"../../../../src/unreal/unreal-fpv-wt2/Saved/Screenshots/qa915_ta_finished.png"`. That path is
relative to the process base dir (`/sdb-disk/UE_5.8/Engine/Binaries/Linux/`), not to the project or to
the MCP client's working directory, so a client cannot open it as given. I rebuilt the absolute path
by hand for every capture (about 12).

Same root as `B-relative-project-dir-paths-doubled` (`FPaths::ProjectDir()` is relative on this layout),
but on the output side: the verb reports the relative engine form instead of converting it with
`FPaths::ConvertRelativePathToFull`.

**Expected:** `path` is absolute. If needed, keep the relative form in a second field.

## History
- `#1-relative-output-path` `OPEN` reporter - "Filed from the PDS QA #915 repro. All editor.screenshot responses carried a ../../../../-relative path; absolute paths were rebuilt by hand."
