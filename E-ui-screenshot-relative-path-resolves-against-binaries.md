---
id: E-ui-screenshot-relative-path-resolves-against-binaries
title: "ui.screenshot writes a relative `path` under Engine/Binaries/<Platform> instead of the project, unlike every other filepath param"
status: OPEN
severity: Low
category: ergonomic
tags: [ui, ui.screenshot, paths, filepath, consistency]
encounters: 1
lastSeen: 2026-10-03T00:00:00Z
rice: [1, 1, 1, 1]
priority: 8
---

# ui.screenshot relative `path` lands under the engine binaries dir

`ui.screenshot` passes its caller-supplied `path` (declared `filepath`, `Handlers/UI/UiHandler.cpp:206`)
straight into `PinWrightScreenshotUtils::MakeUiScreenshotPath` (`Utils/ScreenshotUtils.cpp`), which joins it
with the filename and only converts it to full. A relative `path` such as `MyShots/Sub` is therefore written
under the process base dir: `/sdb-disk/UE_5.8/Engine/Binaries/Linux/MyShots/Sub/frame.png` on this host,
inside the engine tree and outside the project.

Every other verb that takes a caller-supplied filesystem path resolves a relative one against the project
through `ResolveProjectFilePath` (`Utils/PathUtils.h:101-108`: "A relative path is project-relative
(\"Saved/x.png\"), never CWD- or BaseDir-relative"), including `image.*` output dirs, whose
`ResolveOutputDir` cites ui.screenshot as its precedent (`Handlers/Image/ImageOps.h:210`).

Since `E-screenshot-path-relative-to-binaries` `#3`, the echoed `screenshotPath` is absolute and
`docs/wiki-src/ui.md` documents the base-dir resolution, so the file is findable. The write location still
breaks the plugin's own convention.

**Expected:** resolve `path` with `ResolveProjectFilePath` before composing, so `path:"Saved/Shots"` lands in
`<Project>/Saved/Shots`. This is a behaviour change (CHANGELOG). Update `ui.md` and the third block of
`PinWright.editor.screenshot.OutputPathIsAbsolute` (`Tests/EditorOps/TestUiScreenshotFilenameExtension.cpp`),
which pins the base-dir resolution today.

## History
- `#1-found-in-review` `OPEN` reviewer - "Found while reviewing E-screenshot-path-relative-to-binaries #3. Source-read only, not reproduced live: a relative ui.screenshot `path` resolves against FPlatformProcess::BaseDir() (Unix NormalizeFilename -> ConvertRelativePathToFull; Windows cwd = BaseDir), not against the project like ResolveProjectFilePath callers."
