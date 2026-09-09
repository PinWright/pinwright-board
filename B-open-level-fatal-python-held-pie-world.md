---
id: B-open-level-fatal-python-held-pie-world
title: "editor.open_level is fatal after any PIE session whose objects were touched through python.execute — FPyReferenceCollector keeps the dead PIE world alive and Map_Load's unconditional leak check appErrors"
status: OPEN
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

## History
- `#1-py-wrapper-holds-garbage-pie-world` `OPEN` reporter — Observed on UE 5.8 / `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`. `editor.open_level` after a PIE session whose objects had been read through `python.execute` killed the editor with `World Memory Leaks: 1 leaks objects and packages` at `EditorServer.cpp:1951`, the reference-chain root naming `FPyReferenceCollector::AddReferencedObjects((Garbage) World /Temp/Untitled_3.L_Arena)`. Retried with `scope: "private"` and it made no difference, which is consistent with the scope flag being a `sys.modules` scrub rather than a wrapper-metadata drop (`PythonExecuteHandler.cpp:121,131`). Verified in source that the plugin's load site is `LevelHandler.cpp:270` (`FEditorFileUtils::LoadMap`) and that nothing on that path flushes the interpreter or collects first. Filed separately from `B-level-load-dirty-world-fatal` after reading it and its supplement in full: same assertion class, different holder (garbage PIE world held by Python vs dirty content package skipped by `UnloadPackages` policy), different check line, and the fix shipped there (`saveDirtyTargetWorld` + dirty-package precondition) provably cannot detect this holder. Marked `costly`: an editor kill and a restart, with the PIE state and everything unsaved in it lost. Severity `High` rather than `Critical` in deference to the same standing reprioritization applied elsewhere on this board; a fixer who wants to rate it by the impact rubric (editor crash) should raise it.
