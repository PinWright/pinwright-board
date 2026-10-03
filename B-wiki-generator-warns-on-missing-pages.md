---
id: B-wiki-generator-warns-on-missing-pages
title: "Wiki generator logs a LogStreaming \"Failed to read file\" warning for every page not yet on disk"
status: OPEN
severity: Low
category: bug
tags: [wiki, logging, startup]
encounters: 1
lastSeen: 2026-09-30T09:52:27Z
rice: [2, 1, 1, 1]
priority: 17
---

# Wiki generator logs a LogStreaming "Failed to read file" warning for every page not yet on disk

`WikiDiskGenerator_PublishChangedBytes` (`Source/PinWright/Private/Catalog/WikiDiskGenerator.cpp:572-594`) compares the desired bytes against the file already on disk before writing, by calling `FFileHelper::LoadFileToArray(ExistingBytes, *Path)` at line 579 with default flags. The engine logs `LogStreaming: Warning: Failed to read file '%ls' error.` whenever the reader cannot be created and `FILEREAD_Silent` is absent (`C:\UE_5.8\Engine\Source\Runtime\Core\Private\Misc\FileHelper.cpp:47-52`). A missing file is the normal case for a page that does not exist yet, so the check emits a spurious warning.

Both publish sites route through this function: page files (`:794`) and the ownership manifest (`:861`). The warning therefore fires once for every page that is new since the last generation, and for every page on a clean output directory. It is not limited to a first boot: any boot after new methods or overlay pages land logs one warning per new page. Agents that grep editor logs for `Warning:` see these lines as noise, and they are indistinguishable from real read failures.

Evidence: `Saved/Logs/cleanup_editor.log` (host project) contains 36 × `LogStreaming: Warning: Failed to read file 'X:/src/unreal/unreal-fpv-dev/Saved/PinWright/wiki/<page>.md' error.` (e.g. `ai.get_runtime_state.md`, `animation.authoring.get_curve_keys.md`), all at editor startup. Later boots logged 0.

**Workaround:** Ignore `LogStreaming` "Failed to read file" warnings whose path is under `Saved/PinWright/wiki/`.

**Fix:** Pass `FILEREAD_Silent` at `WikiDiskGenerator.cpp:579` (`LoadFileToArray(ExistingBytes, *Path, FILEREAD_Silent)`). A missing or unreadable file already falls through to the write path, and a write failure is reported by the caller, so the read needs no warning of its own. The one shared function covers both call sites. Add a test that a publish to an absent path logs no `LogStreaming` warning.

## History
- `#1-new-page-read-warning` `OPEN` reporter — Verified from source: `PublishChangedBytes` pre-reads the target with `LoadFileToArray` and no `FILEREAD_Silent` (`WikiDiskGenerator.cpp:579`), and the engine warns on a missing file (`FileHelper.cpp:50-52`). Observed 36 warnings for new wiki pages at startup in `Saved/Logs/cleanup_editor.log`, and 0 on later boots. Low: this is log noise only, and the generated output is correct.
