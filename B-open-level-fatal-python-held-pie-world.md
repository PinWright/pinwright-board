---
id: B-open-level-fatal-python-held-pie-world
title: "editor.open_level is fatal after any PIE session whose objects were touched through python.execute — FPyReferenceCollector keeps the dead PIE world alive and Map_Load's unconditional leak check appErrors"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, open-level, level-load, python-execute, pie, world-memory-leaks, editor-crash, map-load, garbage-world]
encounters: 1
costly: 1
lastSeen: 2026-09-09T10:00:00Z
---

# A PIE session plus one `python.execute` makes the next `open_level` a process kill

Sequence, reproduced on UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`:

1. run PIE,
2. touch any PIE object through `python.execute` (reading a property off a live actor is
   enough),
3. end PIE,
4. `editor.open_level` -> the editor dies:

```
Fatal error: [File:...\Editor\UnrealEd\Private\EditorServer.cpp] [Line: 1951]
World Memory Leaks: 1 leaks objects and packages. See The output above.

  (reference chain root)
  FPyReferenceCollector::AddReferencedObjects((Garbage) World /Temp/Untitled_3.L_Arena)
```

The retained object is the **dead PIE world**, already marked Garbage, and the only
thing keeping it reachable is the Python plugin's own reference collector: UE's Python
wrapper metadata holds every `UObject` a script has ever wrapped, and nothing on the
PinWright side drops those wrappers when PIE tears down. `editor.open_level` then reaches
`FEditorFileUtils::LoadMap`
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Level\LevelHandler.cpp:270`),
`Map_Load` runs its post-`Cleanse` leak assertion, finds the world still referenced, and
`appError`s. The check is unconditional — there is no engine-side flag to make it
tolerant — so the caller has no way to survive it once the reference exists.

`scope: "private"` does **not** protect. That mode restores `sys.modules` after the call
(`PythonExecuteHandler.cpp:121,131`), which is a *module* scrub; the wrapper metadata the
reference collector walks is per-object and lives in the interpreter regardless of scope.

## Why this is a separate defect from the dirty-world family

`B-level-load-dirty-world-fatal` (IN-REVIEW, primary) and its supplement
`B-level-load-dirty-world-memory-leak-fatal` describe the same *engine assertion* reached
by a different route: a **dirty, non-garbage** world package that
`UPackageTools::UnloadPackages` refuses to unload by policy
(`PackageTools.cpp:381`), fixed there by a dirty-package precondition plus a
`saveDirtyTargetWorld` opt-in. That guard cannot see this case at all:

- the retained world is a `/Temp/Untitled_*` **PIE** world, not a content package;
- it is already **Garbage**, so no dirty-package sweep and no `IsDirty()` probe reports
  it;
- the holder is the Python reference collector, not the package's `RF_Standalone` flag.

A `level.load` / `open_level` that passes the shipped dirty-world precondition still dies
here. Their line number differs too (`EditorServer.cpp:1951` here vs `:2544` there), i.e.
a different check site in `Map_Load`.

Also distinct from `B-python-execute-reentrant-gc-crash` (a collect running *with a live
Python frame on the stack*); here the script finished long before the load.

## Proposed fix

Before `FEditorFileUtils::LoadMap` in the `editor.open_level` / `level.load` path:

1. run a Python-side flush — `gc.collect()` in the embedded interpreter, plus dropping
   whatever wrapper caches the plugin can reach — then `CollectGarbage`;
2. re-check for a surviving `/Temp/Untitled_*` world (or, generally, any `UWorld` with
   `EWorldType::PIE` still resident);
3. if one survives, **refuse** with a typed error naming the world and the remedy, rather
   than issuing a `MAP LOAD` into a condition the engine treats as fatal. A refused call
   costs one round trip; the current behaviour costs every agent attached to the editor
   their session.

Document the interaction on both the `python.execute` and `editor.open_level` wiki pages:
touching PIE objects from Python arms a later map load. Today nothing warns.

**Workaround:** after any PIE session in which `python.execute` touched a PIE object,
restart the editor before the next `open_level` / `level.load`. There is no in-editor
remedy — the reference is not reachable from any PinWright verb.

## Fix

**IMPLEMENTED as the same pre-swap survivor probe that closes
`B-dirty-world-guard-only-covers-requested-map`** — one probe, because it is one check site:
`UEditorEngine::CheckForWorldGCLeaks` (`EditorServer.cpp:1911-1957`, the `:1951` fatal in this
report), reached from `EditorDestroyWorld`. The probe's remedy step is what this ticket needs.

`FPyReferenceCollector` is private to the Python plugin (`PyReferenceCollector.h` is under its
`Private/`, and the class carries no export macro), so the plugin cannot call
`PurgeUnrealObjectReferences` directly. The public route is the delegate the Python plugin itself
listens on: `FEditorSupportDelegates::PrepareToCleanseEditorObject`
(`PythonScriptPlugin.cpp:1259` -> `:2183`), whose handler is exactly
`PurgeUnrealObjectReferences(InObject, /*bIncludeInnerObjects=*/true)`. `EditorDestroyWorld`
broadcasts the same delegate for the world it is about to tear down (`:2043`) — the dead PIE
world just is not one of them, which is the whole defect.

So `ProbeResidentWorldSurvivors` broadcasts it for every world the leak check would count (a
PIE world whose `FWorldContext` `EndPlayMap` destroyed is one; a live PIE world still has its
context and is untouched), then calls `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, full purge)`.
The interpreter's own `gc.collect` runs first inside that, off
`FCoreUObjectDelegates::GetPreGarbageCollectDelegate`, which the Python plugin hooks
(`PythonScriptPlugin.cpp:1397` -> `OnPreGarbageCollect`). So the reported sequence normally now
**loads successfully** rather than refusing: the wrappers are dropped and the world is reclaimed.
A world that still survives is named in a `DIRTY_WORLD_BLOCKS_MAP_SWAP` refusal carrying its
path, world type, garbage flag and shortest reference chain, instead of killing the process.

Covers `editor.open_level` and `editor.open_asset` on a World through their cross-dispatch to
`level.load` (`LevelHandler.cpp` is the single `FEditorFileUtils::LoadMap` call site), plus
`level.create`, which reaches the same fatal through `NewMap`.

The interaction is documented on `docs/wiki-src/python.md` ("Calls that crash the editor") and
`docs/wiki-src/level.md`.

## History
- `#1-py-wrapper-holds-garbage-pie-world` `OPEN` reporter — Observed on UE 5.8 / `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`. `editor.open_level` after a PIE session whose objects had been read through `python.execute` killed the editor with `World Memory Leaks: 1 leaks objects and packages` at `EditorServer.cpp:1951`, the reference-chain root naming `FPyReferenceCollector::AddReferencedObjects((Garbage) World /Temp/Untitled_3.L_Arena)`. Retried with `scope: "private"` and it made no difference, which is consistent with the scope flag being a `sys.modules` scrub rather than a wrapper-metadata drop (`PythonExecuteHandler.cpp:121,131`). Verified in source that the plugin's load site is `LevelHandler.cpp:270` (`FEditorFileUtils::LoadMap`) and that nothing on that path flushes the interpreter or collects first. Filed separately from `B-level-load-dirty-world-fatal` after reading it and its supplement in full: same assertion class, different holder (garbage PIE world held by Python vs dirty content package skipped by `UnloadPackages` policy), different check line, and the fix shipped there (`saveDirtyTargetWorld` + dirty-package precondition) provably cannot detect this holder. Marked `costly`: an editor kill and a restart, with the PIE state and everything unsaved in it lost. Severity `High` rather than `Critical` in deference to the same standing reprioritization applied elsewhere on this board; a fixer who wants to rate it by the impact rubric (editor crash) should raise it.
- `#2-pre-swap-purge-and-survivor-probe` `IN-REVIEW` developer — Added a pre-swap Python-wrapper purge plus survivor probe in `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.{h,cpp}`: `ProbeResidentWorldSurvivors` enumerates the worlds `UEditorEngine::CheckForWorldGCLeaks` would count (`IsWorldCountedByLeakCheck`), broadcasts `FEditorSupportDelegates::PrepareToCleanseEditorObject` for each — the only public route to `FPyReferenceCollector::PurgeUnrealObjectReferences`, since that class and its header are private to the Python plugin — then runs `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, full purge)`, whose pre-GC delegate drives the interpreter's own `gc.collect`, and re-enumerates. The reported sequence therefore normally loads instead of dying; a world that still survives is refused with `DIRTY_WORLD_BLOCKS_MAP_SWAP` and a `survivingWorlds[]` payload naming its path, world type, garbage flag and shortest `FReferenceChainSearch` root path. Wired in `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` at the single `FEditorFileUtils::LoadMap` site (so `editor.open_level` and `editor.open_asset` on a World are covered through their cross-dispatch) and before `NewMap` in `level.create`. Documented in `Plugins/PinWright/docs/wiki-src/python.md` and `level.md`. Regression test `PinWright.core.map_swap_guard.SurvivorProbeNamesStronglyHeldDeadWorld` in `Plugins/PinWright/Source/PinWright/Private/Tests/Core/TestMapSwapWorldSurvivorProbe.cpp` holds a contextless, non-kept-type world with a strong reference — the stand-in for the wrapper registry, since a real PIE world cannot be reproduced inside an automation test without a PIE session — and asserts the probe names it; counterfactual: revert the probe and `IsBlocked` plus the by-path assertion fail, because that world is exactly what the engine counts and fatals on. `PinWright.core.map_swap_guard.SurvivorProbeIgnoresDirtyResidentWorldPackage` pins the other side, that a dirty resident world package is never refused. No build or runtime test was run.
- `#3-verifier-corrections` `IN-REVIEW` developer — Follow-up on the shared probe. The purge step is unchanged (`PrepareToCleanseEditorObject` broadcast reaching `FPyReferenceCollector::PurgeUnrealObjectReferences`, then the collect whose pre-GC delegate drives the interpreter's `gc.collect`), but it is now guarded and wider: `ProbeResidentWorldSurvivors` early-outs with `bProbeUnavailable` when `IsGarbageCollecting()` or `IsLoading()` rather than running a `CollectGarbage` that `GarbageCollection.cpp` asserts on, flushes async loading and asset compilation first, and takes `bTransactionBufferWillBeCleared` so the undo-buffer exemption applies only on `Map_Load` paths. Coverage extended through the new shared `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/MapSwapGuardRefusal.h` to `level.create`'s already-exists console `Open <path>` branch and to `lighting.create_lighting_enabled_level`; an unavailable probe refuses with the retryable `EDITOR_NOT_READY` instead of swapping blind. Docs corrected: this report's "the check is unconditional — there is no engine-side flag to make it tolerant" is wrong on 5.8. `EditorServer.cpp:1949` gates the severity on `Editor.CheckForWorldGCLeaksAreFatal` (default true); setting it false degrades the kill to a logged Error while leaking the world — an operator escape hatch, not a remedy, and `docs/wiki-src/python.md` now says so. The test file makes one `ProbeResidentWorldSurvivors` call in total (the suite keeps GC centrally scheduled), with the classifier table moved to `PinWright.core.map_swap_guard.LeakCheckClassifierIgnoresDirtyAndKeptWorlds`, which now includes the contextless-PIE-world case this ticket is about and collects nothing. No build or runtime test was run.
