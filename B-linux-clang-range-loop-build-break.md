---
id: B-linux-clang-range-loop-build-break
title: "PinWright does not compile on the UE 5.8 Linux clang toolchain: DataTableAuthoringHandler.cpp:496 iterates FJsonObject::Values as TPair<FString, ...>, which is -Werror,-Wrange-loop-construct now that JSON object keys are UE::TSharedString"
status: OPEN
severity: High
category: bug
tags: [build, linux, clang, werror, range-loop-construct, json, data-table, ue-5.8]
encounters: 2
costly: 1
lastSeen: 2026-09-30T15:40:00Z
---

# Linux build breaks on a range-for over FJsonObject::Values

UE 5.8 Linux (clang, UBT `-Werror` defaults), host project PDS at `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin
`2580e7f4`. Building the editor target with the plugin fails:

```
Plugins/PinWright/Source/PinWright/Private/Handlers/DataTable/DataTableAuthoringHandler.cpp:496:64: error: loop variable 'Entry' of type 'const TPair<FString, TSharedPtr<FJsonValue>> &' (aka 'const TTuple<FString, TSharedPtr<FJsonValue>> &') binds to a temporary constructed from type 'ItElementType &' (aka 'TTuple<UE::TSharedString<char16_t>, TSharedPtr<FJsonValue>> &') [-Werror,-Wrange-loop-construct]
...
Result: Failed (OtherCompilationError)
```

The line, unchanged at `2580e7f4` and at `origin/master` `adb239fd`:

```cpp
// DataTableAuthoringHandler.cpp:496
for (const TPair<FString, TSharedPtr<FJsonValue>>& Entry : Value->AsObject()->Values)
```

On UE 5.8 the key type of `FJsonObject::Values` is `UE::TSharedString<char16_t>`, not `FString`, so each
element is converted into a temporary `TPair<FString, ...>` and the reference binds to it. Clang treats that
as `-Wrange-loop-construct`, which the Linux toolchain promotes to an error. The line came in with
`29e9d445` ("Fix product defects found while verifying the suite").

**Workaround:** add `-CompilerArguments=-Wno-error=range-loop-construct` to the UBT command line. The build
then passes with one warning at the same line.

**Fix:** use the pattern the rest of the plugin already uses for this, e.g. `Utils/JsonUtils.cpp:576-578`:
`for (const auto& Entry : Value->AsObject()->Values)` with `const FString Key = EARGCompat::JsonKeyToString(Entry.Key);`,
and use `Key` in place of `Entry.Key` in the loop body (`*Entry.Key` at `:502`, `Entry.Key` at `:504`). This is the
only `TPair<FString, TSharedPtr<FJsonValue>>&` loop left in `Source/`.

## History
- `#1-linux-build-breaks` `OPEN` reporter - Hit while building PDS on UE 5.8 Linux with plugin `2580e7f4` (the project's PinWright submodule bump). The first build (about 31 minutes) failed on this one error; a second build with `-Wno-error=range-loop-construct` succeeded. Rated High: every Linux clang build of the plugin fails until someone finds the flag, and there is no in-plugin workaround. Costly: one full editor build round lost.
- `#2-still-breaks-at-master-head` `OPEN` reporter - Still breaks at plugin HEAD `27b2d04d` (origin/master, the only remote branch; `DataTableAuthoringHandler.cpp` unchanged since `29e9d445`). Full `PDSEditor Linux Development` build of `/sdb-disk/src/unreal/unreal-fpv-wt1` failed on the same `:496:64` `-Werror,-Wrange-loop-construct` error (`Result: Failed (OtherCompilationError)`, 103 s); it is the only `TPair<FString, ...>` loop over `AsObject()->Values` in `Source/`. Earlier the same day, the QA #830 fix work had to build only the touched modules (`-Module=GameSettings+PDSGame+App`) to get around it. Fix: iterate with `const auto& Entry`.
