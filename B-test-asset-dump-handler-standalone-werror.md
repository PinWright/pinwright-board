---
id: B-test-asset-dump-handler-standalone-werror
title: "TestAssetDumpHandler.cpp fails -Werror when compiled outside its unity blob"
status: DONE
severity: Low
category: bug
tags: [build, tests, unity-build, werror]
encounters: 1
costly: 0
lastSeen: 2026-10-03T00:00:00Z
---

# TestAssetDumpHandler.cpp fails -Werror when compiled outside its unity blob

`PinWright.asset.dump.AsyncFolderDump.CacheVersionInvalidation` takes the first key of a JSON object with `for (const TPair<FString, TSharedPtr<FJsonValue>> Pair : AspectVersions->Values) { FirstAspect = Pair.Key; break; }` (`Source/PinWright/Private/Tests/Utility/TestAssetDumpHandler.cpp`, about line 1579 in the working tree, line 1481 at 7230b41d, present since 8748c637). The module compiles with `-Wall -Werror -Wunreachable-code-aggressive` (Module.PinWright.*.cpp.o.rsp), and a standalone compile of the file with those exact flags fails: `error: loop will run at most once (loop increment never executed) [-Werror,-Wunreachable-code-loop-increment]`.

The normal build is not affected today: the file is compiled inside `Module.PinWright.80.cpp`, and `clang -fsyntax-only` on that unity TU with the same rsp reports no diagnostic. Any non-unity compile of the file breaks: UBT `-SingleFile` (the documented per-TU compile check), `-DisableUnity`, adaptive non-unity, or a unity regrouping that changes what the diagnostic sees. It also makes the fastcheck syntax check report a false failure for every developer who touches this file.

**Fix:** take the first key without a loop that breaks immediately, e.g. `TArray<FString> Keys; AspectVersions->Values.GetKeys(Keys); if (Keys.Num() > 0) FirstAspect = Keys[0];` or `auto It = AspectVersions->Values.CreateConstIterator(); if (It) FirstAspect = It.Key();`.

## History
- `#1-standalone-werror-found` `OPEN` reviewer — Found while reviewing `B-dump-folder-unbounded-asset`: fastcheck of TestAssetDumpHandler.cpp fails on this pre-existing loop; the unity TU containing it compiles clean with the same flags, so only standalone compiles break.
- `#2-first-element-if` `IN-REVIEW` developer — Replaced the `for … break` loop in `PinWright.asset.dump.AsyncFolderDump.CacheVersionInvalidation` (`Tests/Utility/TestAssetDumpHandler.cpp`) with `if (Values.Num() > 0) FirstAspect = Values.CreateConstIterator()->Key;`, which has the same behaviour. A standalone fastcheck of the file is now clean. Fixed alongside `B-dump-folder-unbounded-asset`.
- `#3-verified-linux` `DONE` tester — Linux, UE 5.8, PinWright `ae877ccc` (fix in commit `de71f65a`). Standalone check: the module's own compile flags (`Module.PinWright.80.cpp.o.rsp` / `PinWright.Shared.rsp`: `-Wall -Werror -Wunreachable-code-aggressive`, shared PCH, `Definitions.PinWright.h`) with `-fsyntax-only` run on the committed `Tests/Utility/TestAssetDumpHandler.cpp` alone, with the engine's clang 20.1.8, gives exit 0 and no diagnostics. The only flag added was `-Wno-unused-command-line-argument`, because the shared rsp carries `-c`. Control: the same command on the pre-fix file (`de71f65a^`) fails with exactly `error: loop will run at most once (loop increment never executed) [-Werror,-Wunreachable-code-loop-increment]` at :1481. Behaviour is unchanged: `PinWright.asset.dump.AsyncFolderDump.CacheVersionInvalidation` passed in b6/run3/full (5827/5827, 0 failed), with no skip marker and not in skipids. The unity build in run2 was clean. Coverage limit: this is a syntax-only standalone pass, not a UBT `-SingleFile` or `-DisableUnity` object build.
