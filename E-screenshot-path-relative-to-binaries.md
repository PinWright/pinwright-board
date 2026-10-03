---
id: E-screenshot-path-relative-to-binaries
title: "editor.screenshot returns path relative to the engine binaries directory (../../../../src/...), which is unusable from the caller's working directory"
status: DONE
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
- `#3-absolute-at-shared-helper` `IN-REVIEW` developer — Fixed at the two shared helpers in `Utils/ScreenshotUtils.cpp` (plugin base `7230b41d`). `MakeScreenshotOutputPath` now roots at `ConvertRelativePathToFull(ProjectSavedDir())`, which covers every caller: `editor.screenshot`, `editor.screenshot_window`, `editor.standalone_status` capture (already converted), `widget.screenshot_designer`, `render.capture_annotated`, z-fighting mask, mesh/actor/asset-preview and open-level captures, and the drive set-of-mark image. `MakeUiScreenshotPath` (`ui.screenshot` `screenshotPath`) had the same defect via `FPaths::MakeStandardFilename`, which rewrites any path under the engine root dir to the `../../../`-relative form; it now returns `ConvertRelativePathToFull`, the same resolution the platform file layer applies to the relative write, so the echo names the file actually written. Docs: `editor.md` (`path` absolute), `ui.md` (`screenshotPath` absolute, relative `path` resolves against the base dir); CHANGELOG Fixed entry. Test `PinWright.editor.screenshot.OutputPathIsAbsolute` (in `Tests/EditorOps/TestUiScreenshotFilenameExtension.cpp`): both helpers return absolute, `..`-free paths equal to the written file; the ui relative-`path` block fails on any layout if reverted, the `MakeScreenshotOutputPath` block fails wherever `ProjectSavedDir()` is relative (this host; logged via AddInfo).
- `#4-verified-linux` `DONE` tester — Verified on Linux, UE 5.8 Vulkan, PinWright ae877ccc on origin/master (commit cc521038). Test: `PinWright.editor.screenshot.OutputPathIsAbsolute`. run3/full (offscreen full suite, 5827/5827 ok, 0 fail) passed them non-skipped: none is in skipids.txt and none carries a PINWRIGHT_ASSERTIONS_SKIPPED marker. The test logged `ProjectSavedDir() on this host: '../../../../src/unreal/unreal-fpv-wt2/Saved/' (relative: yes)`. That is the reporter's exact layout, so the `MakeScreenshotOutputPath` block ran in its failing-on-revert configuration. The Expected item is met: both shared helpers (`MakeScreenshotOutputPath`, and `MakeUiScreenshotPath` for `ui.screenshot`'s `screenshotPath`) return absolute paths with no `..`, each equal to the file actually written. `editor.screenshot` and the other capture verbs publish the first helper's return as `path` unchanged. `editor.md` and `ui.md` document the absolute path. Coverage limit: the test exercises the helpers, not an `editor.screenshot` RPC round-trip. The verb-to-helper wiring was not changed.
