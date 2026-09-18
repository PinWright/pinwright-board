---
id: B-dump-root-double-prefixes-project-dir
title: "ResolveDumpRoot re-prefixes ProjectDir onto an already project-relative outRoot"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, paths, linux, tests]
encounters: 1
---

# ResolveDumpRoot re-prefixes ProjectDir onto an already project-relative outRoot

`AssetDumpWriter::ResolveDumpRoot` (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp`
~:28-35) treats *any* relative `OutRoot` as relative to the project and prepends
`FPaths::ProjectDir()`:

```cpp
else if (FPaths::IsRelative(Root))
{
    Root = FPaths::ProjectDir() / Root;
}
```

That assumes `ProjectDir()` is absolute. It is on Windows. It is not when the
project lives outside the engine tree on a Linux host: there `ProjectDir()` —
and therefore everything derived from it, including `FPaths::ProjectIntermediateDir()`
— comes back as a long `../../../../`-style relative path pointing from the engine
`Binaries` directory across to the project. Callers that build an `outRoot` out of
`ProjectIntermediateDir()` (the normal, correct way to get a scratch dir) hand
`ResolveDumpRoot` a string that is already project-relative, get a second
`ProjectDir()` glued on the front, and the doubled `../` run walks past the
filesystem root. `ConvertRelativePathToFull` then collapses it to an absolute path
rooted at `/`.

On the reporting host the doubled `../` run consumed the leading segments of the
checkout path, so the dump root resolved to a short path directly under `/`
instead of under the checkout, and every write failed with `EACCES` against the
read-only root directory.

**Impact:** 37 `PinWright.asset.dump.*` tests fail on any Linux host whose project
sits outside the engine tree — every test whose fixture writes under a
`ProjectIntermediateDir()`-derived `outRoot`. It is not test-only: any MCP caller
passing a relative `outRoot` on such a host dumps to the wrong place or fails
outright. Completely invisible on Windows, where `ProjectDir()` is absolute and the
double prefix is a no-op, which is why it survived the Win64 suite.

**Evidence:** `Saved/PinWright/test-runs/wrapup_2026-09-17/automation.log` in the
host project — 38 failures, 37 of them `asset.dump`, and 6004 lines of
`LogUnixPlatformFile: Warning: create dir('/src/') failed: errno=13 (Permission denied)`.
(Host-project path; a fresh clone will not have this log — re-measure.)

**Workaround:** pass an absolute `outRoot` (wrap it in
`FPaths::ConvertRelativePathToFull`) — which is what the one test that does not
fail already does.

**Fix:** normalise before deciding. Resolve `OutRoot` to full *first*
(`ConvertRelativePathToFull` against the CWD is still wrong for the CWD-relative
case the existing comment guards against), so the rule has to become: if
`OutRoot` is relative, join it onto `ConvertRelativePathToFull(ProjectDir())`
rather than the raw `ProjectDir()`. That keeps the intended CWD-independence and
stops the double prefix in one line. Worth a unit test that feeds
`ResolveDumpRoot` a `ProjectIntermediateDir()`-derived root and asserts the result
is a prefix of the absolute project dir — it would fail today on Linux and pass on
Windows, which is exactly the asymmetry to lock down.

## History
- `#1-initial-repro` `OPEN` reporter — 37 `PinWright.asset.dump.*` tests fail on a Linux host with the project outside the engine tree; `ResolveDumpRoot` prepends `FPaths::ProjectDir()` to an `outRoot` that is itself project-relative (because `ProjectIntermediateDir()` is relative there), the doubled `../` run walks past `/`, and the dump root collapses to `/src/...` (EACCES). Evidence: `Saved/PinWright/test-runs/wrapup_2026-09-17/automation.log`, `create dir('/src/') failed: errno=13`. No repro on Windows, where `ProjectDir()` is absolute.
- `#2-fixed-callers-and-hardened-resolve` `IN-REVIEW` developer — Contract kept as documented: a relative `outRoot` is PROJECT-relative, and `AssetDumpRootDirectory` (a host may set it to `asset-dumps`) still resolves to `<project>/asset-dumps`. Two changes. (a) `AssetDumpWriter::ResolveDumpRoot` now joins a relative root onto `FPaths::ConvertRelativePathToFull(FPaths::ProjectDir())` instead of the raw `ProjectDir()`, so the join base is absolute and the resolved root never depends on the executable's BaseDir. (b) The real defect was caller-side: 29 test files built their scratch `outRoot` from `FPaths::ProjectIntermediateDir()`, which is BaseDir-relative, not project-relative — every one now wraps it in `FPaths::ConvertRelativePathToFull(...)`. Note for the record: the fix proposed in the original report ("join onto `ConvertRelativePathToFull(ProjectDir())`") is NOT sufficient on its own — `/abs/project` + `../../../../src/...` still collapses past the mount root. The caller fix is the load-bearing half. New unit test `PinWright.utils.asset_dump_writer.ResolveDumpRoot` covers relative-under-absolute-project-dir, absolute-untouched, and the failure direction (a `../` root resolves beside the project, not at the filesystem root). Evidence: run `Saved/PinWright/test-runs/dumpfixes_2026-09-18b/automation.log` on the reporting Linux host — all 38 tests that failed in `wrapup_2026-09-17` are green, `create dir('/src/')` lines are gone. Tester still required for DONE.
