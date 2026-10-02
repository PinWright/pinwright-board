---
id: E-screenshot-path-relative-to-binaries
title: "editor.screenshot returns path relative to the engine binaries directory (../../../../src/...), which is unusable from the caller's working directory"
status: OPEN
severity: Medium
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
- `#2-stale-sweep-bump-medium` `OPEN` developer — Severity Low -> Medium. Still reproduces in source at plugin HEAD `10212ee4`: `PinWrightScreenshotUtils::MakeScreenshotOutputPath` (`Utils/ScreenshotUtils.cpp:353-378`) builds `FPaths::ProjectSavedDir() / "Screenshots" / <file>` and returns it unconverted, and `editor.screenshot` writes it straight into `path` (`ViewportHandler.cpp:856`, `:912`). On a layout where `ProjectDir()` is relative (same-root Linux checkout), `path` is relative to `Engine/Binaries/Linux` and unusable by the client. `B-relative-project-dir-paths-doubled` (IN-REVIEW) fixed the input side only. Rubric: Medium impact (a readback field the caller cannot use forces a hand-rebuilt fallback on every call: about 12 manual rebuilds in one session), and `editor.screenshot` runs in almost every session, so the reach modifier lifts it out of Low. The same helper also feeds the annotated, mesh-preview and z-fighting captures, so one `ConvertRelativePathToFull` in the helper fixes all of them.
