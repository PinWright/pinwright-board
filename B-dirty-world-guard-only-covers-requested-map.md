---
id: B-dirty-world-guard-only-covers-requested-map
title: "`level.load`'s DIRTY_WORLD_BLOCKS_MAP_SWAP guards only the requested map, while the fatal it prevents is triggered by the outgoing resident world"
status: IN-REVIEW
severity: High
category: bug
tags: [level, shared-editor, crash, guard]
---

# `level.load`'s DIRTY_WORLD_BLOCKS_MAP_SWAP guards only the requested map, while the fatal it prevents is triggered by the outgoing resident world

`level.load` / `editor.open_level` refuse with `DIRTY_WORLD_BLOCKS_MAP_SWAP` when **the map being
requested** is already resident with unsaved changes. The stated mechanism is general:

> to re-read the map the engine has to unload the resident copy, it will not unload a dirty package,
> and `UEditorEngine::Map_Load` then reaches an unconditional `Fatal` ("World Memory Leaks") that
> kills the editor process and every session attached to it

That mechanism does not depend on the dirty package being the *requested* one. Swapping **away**
from a dirty resident world requires unloading that dirty package for exactly the same reason, and
reaches the same `Fatal` — but no guard fires, because the requested map is clean.

## The uncovered case

    resident world : /Game/FPS/Test/T_VFX   dirty, owned by another stream
    requested world: /Game/FPS/Test/T_AI    clean

`level.load {levelPath:'/Game/FPS/Test/T_AI'}` is **not** refused. The caller has to know to run the
check by hand — `EditorLoadingAndSavingUtils.get_dirty_map_packages()` or
`editor.list_dirty_packages` — before every map swap, and to interpret the result themselves.

## Evidence this is the real killer, not a theoretical direction

This checkout has already lost an editor to the outgoing-dirty direction specifically. From the
history on `B-level-load-dirty-world-fatal` (`#2`, filed from this project): the swap that
killed the editor was a **`level.create`** issued while the resident world was dirty — the incoming
world did not exist yet, so "the requested map is dirty" was impossible by construction. The fatal
came from tearing down the dirty *outgoing* world. Every agent attached to that editor lost its
unsaved work.

Today (2026-09-05, 18:14Z) the same shape stopped the AI stream cold: holding the world lock with a
clean `T_AI` to load and a dirty `T_VFX` resident, there was no safe call available. Not refused,
so no diagnostic; the only options were to gamble the whole editor or to give the slot back. The
slot was given back (lock held 60 s, released token-verified). That is the second time in one
session a stream has been blocked by an outgoing dirty world with no verb-level help.

## Why the existing escape hatch does not reach it

`saveDirtyTargetWorld` is scoped to the requested map ("When the requested map is already resident
with unsaved changes, write that ONE package to disk first"). It cannot clear a dirty *outgoing*
world. The only remaining lever is `editor.save_all`, which writes **every** other stream's dirty
packages — in a shared editor that silently commits other agents' work in progress, which is its own
harm and is why this caller refused to use it.

## Ask

1. **Refuse on a dirty resident world too.** When the outgoing world is dirty and must be torn down
   to satisfy the load, refuse with the same class of error and **name the blocking package** in the
   payload, exactly as the requested-map path already does. A refusal costs a retry; the current
   silence costs the process.
2. **Add a discard/save flag scoped to the outgoing map** — the mirror of `saveDirtyTargetWorld`,
   e.g. `saveDirtyResidentWorld` (write that one package and proceed) and/or
   `discardDirtyResidentWorld` (drop its in-memory edits and proceed). Both must be narrower than
   `editor.save_all`: one named package, never a sweep, so a caller can clear its **own** dirty world
   without touching a co-tenant's.
3. Failing either, say in the wiki page that the guard is one-directional and that callers must
   check `get_dirty_map_packages()` before every swap. Right now the page's mechanism paragraph
   reads as general while the implementation is not, which is what makes this easy to walk into.

## Workaround

Before any `level.load` / `editor.open_level` / `level.create` in a shared editor, check
`EditorLoadingAndSavingUtils.get_dirty_map_packages()` (or `editor.list_dirty_packages`). If it is
non-empty and the dirty world is not yours, do not swap: ask its owner to save or discard. Do not
reach for `editor.save_all` to clear it.

## Relationship to existing tickets

Distinct from `B-level-load-dirty-world-fatal` (and its supplement
`B-level-load-dirty-world-memory-leak-fatal`), which report the crash itself and whose fix added
the requested-map guard. This ticket is about the half of the mechanism that guard does not
cover. Fixing it should reuse the same refusal path and payload shape.

## Fix

**IMPLEMENTED as a post-cleanse survivor probe (supersedes everything below).** The tester's
objection in `#4` is the specification and it is correct: package dirtiness is not the retention
mechanism, so a guard built on it both misses the real killers and refuses safe swaps. The
mechanism, read from UE 5.8 source, is `UEditorEngine::CheckForWorldGCLeaks`
(`EditorServer.cpp:1911-1957`), which `EditorDestroyWorld` calls after its `Cleanse`. It walks
every resident `UWorld` and counts one as a leak when it is not the world being kept, its
`WorldType` is not `Inactive`/`EditorPreview`/`GamePreview`, and `UEngine::WorldHasValidContext`
is false. `:1951` then logs `World Memory Leaks` at Fatal. Nothing in that predicate reads a dirty
flag — this is a second check site from the one the requested-map guard covers (`:2544`).

`PinWrightMapSwapGuard::ProbeResidentWorldSurvivors` reproduces that enumeration as a refusable
precondition instead of a fatal: enumerate the counted worlds; broadcast
`FEditorSupportDelegates::PrepareToCleanseEditorObject` for each (the same delegate
`EditorDestroyWorld` broadcasts, and the only public route to
`FPyReferenceCollector::PurgeUnrealObjectReferences`); `CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS,
full purge)`, whose pre-GC delegate drives the interpreter's own `gc.collect`; re-enumerate. What
is still standing is what the engine is about to fatal on, and only that refuses. When nothing is
counted the probe costs one `UWorld` iteration and does not collect at all.

Two exemptions keep it honest before the teardown rather than after it: a streaming sublevel whose
owner still has a context (borrowed from `UEngine::CheckAndHandleStaleWorldObjectReferences`,
`UnrealEngine.cpp:17388` — a different predicate on a different path, so it can only hide a
survivor, never invent one), and a world reachable only through a holder the engine empties before
its own leak check. That second one is the false-refusal class the returned patch produced in a
different form: the undo buffer keeps the actors of a hand-torn-down world, and an actor's outer
chain reaches that world. **The two holders do not have the same reach, and it is entry-point
dependent.** The selection sets are cleared inside `EditorDestroyWorld` itself, so they are always
exempt. `ResetTransaction` is not: `Map_Load` calls it at `EditorServer.cpp:2456`, *before*
`EditorDestroyWorld` at `:2480`, but `NewMap` calls `EditorDestroyWorld` at `:2207` and
`ResetTransaction` only at `:2256` — after `CheckForWorldGCLeaks` has already fired. So the undo
exemption is passed per call site (`bTransactionBufferWillBeCleared`): true on the `Map_Load`
paths, false on the `NewMap` ones, where exempting it would hand back the crash.

The probe refuses to run rather than guessing when a collect would itself be illegal
(`IsGarbageCollecting()`, or `IsLoading()` — `CollectGarbage` asserts `check(!IsLoading())`), and
flushes async loading and asset compilation before collecting, following
`Handlers/Asset/AssetDumpSweepGc.cpp`. That case answers the retryable `EDITOR_NOT_READY`, never a
false all-clear.

Wired into every swap entry point through one shared inline refusal
(`Handlers/Level/MapSwapGuardRefusal.h`): `level.load` (covering `editor.open_level` and
`editor.open_asset` on a World, both of which cross-dispatch to it), `level.create` on **both** its
exits — the `NewMap` branch and the already-exists branch that issues a console `Open <path>`,
which is `Map_Load` by another spelling and previously had no guard at all — and
`lighting.create_lighting_enabled_level`, the other `NewMap` caller in the tree. A `level.create`
is the swap this ticket's own evidence section records as having killed an editor.

**What this does NOT cover, and must not be read as covering: the outgoing world itself.** At probe
time it still owns its `FWorldContext`, so the engine's predicate excludes it and so does this one;
it becomes a candidate only after `EditorDestroyWorld` has cleared the context and run `Cleanse`,
which is inside the call being decided. The engine checks it separately in the same pass (the
`WorldPackage` half, `EditorServer.cpp:1932-1945`). A pre-flight cannot answer that without
performing the teardown, so a green probe means "no *other* dead world is resident", not "this swap
is safe". Stated in the header and in `docs/wiki-src/level.md` as well as here, because the
outgoing direction is this ticket's headline and leaving it implied would overstate the fix.

One wording correction that travelled with the fix: the fatal is **not** unconditional on 5.8.
`EditorServer.cpp:1949` gates it on `Editor.CheckForWorldGCLeaksAreFatal` (default true; false
degrades to a logged Error and leaks the world). The header, the refusal message and the wiki now
name that cvar as an operator lever instead of claiming there is none.

Deliberately NOT implemented: `saveDirtyOutgoingWorld` / `discardDirtyOutgoingWorld`. They were
asked for on the premise that a dirty outgoing package is what kills the load. It is not, so
saving or discarding one buys no safety, and shipping a knob that implies it does would be worse
than shipping nothing. A caller who wants to write their own dirty package still has `level.save`
and `asset.save`. `E-orphan-world-no-discard-verb` is where a discard verb belongs.

---

**Superseded, kept for history — the withdrawn wave-10 patch and the return that rejected it.**

**RETURNED — the current patch is not a valid fix.** UE5.8 `EditorDestroyWorld` clears the
outgoing world's teardown-owned keep flags, destroys the world, runs `Cleanse`, and only then
fatals if the old `UWorld` or its package actually survived garbage collection. Package dirtiness
is not that retention mechanism. The current implementation therefore creates false refusals by
treating every dirty outgoing package as fatal. It also omits the requested scoped save/discard
options, uses a host-project/active-world-dependent handler test, and relies on a source-text test
for part of the claimed contract. The ticket remains open until the guard identifies actual
post-teardown survivors and the remedies are implemented with measured per-package results.

**PARTLY TRUE — core guard fixed; per-resident save/discard opt-in remains unimplemented.** The
root cause was that `level.load` only probed the package named by `levelPath`. It never inspected
the outgoing editor world's loaded streaming-level, map-build-data, or external-object packages,
even though `Map_Load` unloads those packages during the swap and can reach the same fatal when one
is dirty. The handler now performs a last-moment current-world package probe at the safe point,
merges those names with the requested-target result, and refuses with the existing
`DIRTY_WORLD_BLOCKS_MAP_SWAP` code plus `dirtyPackages`, `dirtyCount`, `blockingPackage`, and
`residentWorldDirty`. `dirtyPackages` is sorted; `blockingPackage` remains the primary blocker,
preserving the requested target when it is still dirty and otherwise naming the first current-world
package.

The current-world probe enumerates that package set directly and intentionally does not call
`FEditorFileUtils::GetDirtyWorldPackages`: UE5.8's helper can mark map-build-data or newly-created
world packages dirty while enumerating, which would make this safety precondition mutate the editor.

Changed files:

- `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.h`
- `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp`
- `Plugins/PinWright/Source/PinWright/Private/Tests/Core/TestMapSwapDirtyWorldGuard.cpp`
- `Plugins/PinWright/Docs/wiki-src/level.md`

Regression test added: `PinWright.level.load.DirtyCurrentSublevelRefusesWithPackageList`. It
attaches one on-disk map as a dirty streaming sublevel, selects a different clean on-disk target,
and invokes the production `level.load` handler without reaching `LoadMap`. The test was statically
reviewed but not built or run here. Structural regression added:
`PinWright.core.map_swap_guard.CurrentWorldFilterExcludesNewlyCreatedPackages`, which ratchets the
UE5.8 `PKG_NewlyCreated` exclusion in the current-world candidate filter and asserts that the
current-world probe does not call the mutating engine helper.

Deliberately unchanged: `saveDirtyTargetWorld` still saves only the requested resident target
package, re-probes it, and then loads if the block clears. It does not save or discard packages
owned by the outgoing current world. No `discardChanges`, `saveFirst`, `saveDirtyResidentWorld`, or
`discardDirtyResidentWorld` parameter exists in the current handler/schema. `editor.open_level`
still declares only `levelPath`/`path` and forwards only `levelPath`, so the target-save option is
available on `level.load` but not through that alias. Callers must save/discard the listed resident
packages explicitly or use `editor.save_all` when a full sweep is intended. If the target save
succeeds but a current-world package still blocks the swap, the refusal reports `savedDirtyTargetWorld`
and `savedPackage`; no map swap is performed. Alias forwarding is a deliberately unchanged
follow-up candidate, not part of this fix.

## History
- `#1-current-world-guard` `IN-REVIEW` developer — Added the outgoing current-world dirty-package guard and behavioral regression coverage. Core request/refusal behavior is fixed; the requested per-resident save/discard opt-in is not implemented, so this ticket is PARTLY TRUE. No build or runtime test was run.
- `#2-review-corrections` `IN-REVIEW` developer — Excluded `PKG_NewlyCreated` packages, added the structural filter regression, and corrected mixed-refusal wording/payload documentation so a prior target save is not reported as a no-change operation. Alias forwarding remains an explicit follow-up.
- `#3-read-only-dirty-probe` `IN-REVIEW` developer — Removed the UE5.8 mutating `GetDirtyWorldPackages` call from the current-world precondition, retained direct package enumeration, and extended the structural test to forbid that helper call. No build or runtime test was run.
- `#4-returned-false-dirty-blockers` `OPEN` tester — Returned after source verification: `EditorDestroyWorld` clears teardown-owned keep flags before GC, so ordinary dirty outgoing packages do not cause the fatal. The patch falsely refuses them, lacks `saveDirtyOutgoingWorld` / `discardDirtyOutgoingWorld`, mutates the active host world in its handler test, and uses a source-text structural test instead of a behavioral counterfactual. No build or runtime test was run.
- `#5-withdrawn-invalid-fix` `OPEN` developer — Rejected code, tests, and wiki edits were withdrawn; the plugin tree was restored to f41e086d.
- `#6-post-cleanse-survivor-probe` `IN-REVIEW` developer — Replaced the dirty-package premise with the engine's own leak predicate. `Plugins/PinWright/Source/PinWright/Private/Utils/MapSwapDirtyWorldGuard.{h,cpp}` gains `IsWorldCountedByLeakCheck` (mirrors `CheckForWorldGCLeaks`, `EditorServer.cpp:1918-1927`, plus the live-sublevel and incoming-target narrowings), `ProbeResidentWorldSurvivors` (broadcast `PrepareToCleanseEditorObject` for every counted world, `CollectGarbage` full purge, re-enumerate, name each survivor's path/world type/garbage flag/shortest `FReferenceChainSearch` root path) and `DescribeSurvivorRefusal`. A world reachable only through the undo buffer or the selection sets is exempt — `Map_Load` clears both before its own check, and refusing it is the false-refusal class `#4` returned. `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/LevelHandler.cpp` calls the probe from `level.load` (after the existing target guard, last instruction before `LoadMap`) and from `level.create` (before `NewMap`), refusing through the existing `DIRTY_WORLD_BLOCKS_MAP_SWAP` with a shared `survivingWorlds[]` payload; `Handlers/ErrorCodes.h` and `docs/wiki-src/level.md` document the second refusal shape. Regression tests in `Plugins/PinWright/Source/PinWright/Private/Tests/Core/TestMapSwapWorldSurvivorProbe.cpp`: `PinWright.core.map_swap_guard.SurvivorProbeNamesStronglyHeldDeadWorld` and `PinWright.core.map_swap_guard.SurvivorProbeIgnoresDirtyResidentWorldPackage`. Counterfactuals — revert the probe (or widen the classifier to exempt every world) and the first test's `IsBlocked` and by-path assertions fail, because a contextless `EWorldType::Editor` world held by a strong reference is exactly what the engine counts and fatals on; re-widen the guard to refuse dirty resident world packages and the second test's `TestFalse` on an `Inactive`, dirty, strongly-held world fails. No build or runtime test was run.
- `#7-verifier-corrections` `IN-REVIEW` developer — Acted on the verifier pass. (1) Correctness: the undo-buffer exemption is `Map_Load`-only — `NewMap` calls `EditorDestroyWorld` at `EditorServer.cpp:2207` and `ResetTransaction` only at `:2256`, after `CheckForWorldGCLeaks` — so `ProbeResidentWorldSurvivors` now takes `bTransactionBufferWillBeCleared` and `GatherClearedHolderClasses` gates the `UTransBuffer` class on it; the `USelection` half stays unconditional because `EditorDestroyWorld` deselects internally. (2) Safety: the probe early-outs with `bProbeUnavailable` + reason when `IsGarbageCollecting()` or `IsLoading()` (`CollectGarbage` asserts `check(!IsLoading())`) instead of collecting, and calls `FlushAsyncLoading()` + `FAssetCompilingManager::FinishAllCompilation()` first, per `Handlers/Asset/AssetDumpSweepGc.cpp`; callers refuse that state with the retryable `EDITOR_NOT_READY`, never a false all-clear. (3) Coverage: new shared `Plugins/PinWright/Source/PinWright/Private/Handlers/Level/MapSwapGuardRefusal.h` is now used by `level.load`, by `level.create`'s already-exists branch (a console `Open <path>`, i.e. `Map_Load`, which had no guard), by `level.create`'s `NewMap` branch, and by `lighting.create_lighting_enabled_level` in `Handlers/Environment/LightingHandler.cpp`; the payload builder moved into the guard util as `BuildSurvivorErrorData` so all four report one shape. (4) Docs: "unconditional Fatal" corrected everywhere to the cvar-gated `Editor.CheckForWorldGCLeaksAreFatal` (default true) and named as an operator lever; `DescribeSurvivorRefusal` now states what the probe did to the editor (cleanse broadcast, loading/compilation flush, one full-purge collect) instead of "nothing has been changed"; the outgoing world's structural invisibility to any pre-flight is stated in the header, the Fix section above and `docs/wiki-src/level.md`; the `PersistentLevel->OwningWorld` narrowing is attributed to `CheckAndHandleStaleWorldObjectReferences` and marked as only ever able to hide a survivor. (5) Tests: `Tests/Core/TestMapSwapWorldSurvivorProbe.cpp` now makes exactly ONE `ProbeResidentWorldSurvivors` call in the whole file (both fixtures live at once so the single collect answers both questions), and the discrimination table moved to `PinWright.core.map_swap_guard.LeakCheckClassifierIgnoresDirtyAndKeptWorlds`, which drives `IsWorldCountedByLeakCheck` directly and collects nothing; the probe-unavailable path emits the `PINWRIGHT_ASSERTIONS_SKIPPED` marker through `PinWrightTestSkip::SkipAssertions`. No build or runtime test was run.
