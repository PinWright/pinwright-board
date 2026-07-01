---
id: E-level-load-file-not-found-vs-in-memory-orphan
title: "level.load / editor.open_level emit bare [FILE_NOT_FOUND] for a world that is registered / in-memory but not on disk, driving path-form trial-and-error"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level, load, error-message, file-not-found, in-memory, orphan, registry-vs-disk]
---

# `level.load` says "Level file not found" for a world that *is* loaded/registered

`level.load` and its alias `editor.open_level` gate on disk presence only —
`LevelHandler.cpp:181-186` emits `SendError("FILE_NOT_FOUND", "Level file not
found: <path>")` when both `IFileManager::FileExists` and
`FPackageName::DoesPackageExist` are false. The check never considers whether a
matching `UWorld` / package already exists **in memory** or in the **asset
registry**. So when a world is registered and live in memory but has no `.umap`
(the orphaned-after-create_level state of `B-create-level-saved-true-no-umap`,
but also any registry-listed-but-unsaved package), the agent gets a flat
"Level file not found" — a message that *contradicts what `level.list`
simultaneously reports* (the same path is listed as an existing map) and gives
no hint at the real condition ("registered/loaded in memory, never persisted to
disk").

This is the same conflation class that `E-asset-dump-distinguish-file-missing-
from-load-failed` (DONE) already fixed for `asset.dump` — there, "source file
truly absent on disk" was split out from "present but failed to load" via a
`FPackageName::DoesPackageExist` pre-check and a distinct `ASSET_FILE_MISSING`
code. The `level.*` load path never got the analogous treatment: it has the
opposite blind spot — it reports `FILE_NOT_FOUND` (a *disk* verdict) without
ever telling the caller whether the world is nonetheless present in memory /
registry. The agent cannot tell "this path is garbage" from "this path is a
real, registered, in-memory world that simply hasn't been saved."

## Why it matters — the process friction

Because "Level file not found" reads like a path-resolution failure, an agent
facing a registry-listed-but-unsaved world reasonably suspects it got the
*path form* wrong and burns calls cycling through permutations rather than
recognizing the orphan. In the audited `level.structure` task that is exactly
what happened after `create_level` stranded the in-memory world:

- 5 distinct `level.load` attempts with different path forms — full package
  path `/Game/Maps/Highlands_WP`, object path
  `/Game/Maps/Highlands_WP.Highlands_WP`, bare name `Highlands_WP`, then
  `/Game/Maps/ExampleProjectWelcome` and `/Game/Global/DemoRoom/TestRoom` as
  evict attempts — every `Highlands_WP` form returning `[FILE_NOT_FOUND]`.
- 1 `editor.open_level /Game/Maps/Highlands_WP` → same `[FILE_NOT_FOUND]`
  (`.../Content/Maps/Highlands_WP.umap`).
- 2 `editor.save_all` flushes (`savedCount:0 totalDirty:0`) and 2
  `system.console.search` probes for a save command — all dead ends spawned by
  the unclear error.

The friction note records it directly: "Tried 3 load paths (full/object/bare),
open_level, two save_all flushes, a console save-command search ... none
recovered it." And the call log shows `level.list` "registry lists
/Game/Maps/Highlands_WP but no disk file" *right next to* the `FILE_NOT_FOUND`
loads — the two surfaces openly disagree, and nothing in the error reconciles
them.

A diagnostic that distinguished the disk-absent case from the
in-memory/registry-present case ("no `.umap` on disk for '<path>', but a world
with this name is loaded in memory and was never saved — it cannot be loaded
from disk; save or discard it first") would have collapsed the entire
path-form flail into one read. This is the ergonomic angle the
`B-create-level-saved-true-no-umap` bug ticket does not cover: that ticket
fixes the *cause* (false `saved:true` + orphan trap); this ticket asks the
`level.load`/`open_level` *error message* to stop misdescribing the
registry-vs-disk mismatch, which helps the broader class (any
registered/in-memory-but-unsaved world, not just the create_level orphan).

## Related no-op surface

The same handler's already-loaded short-circuit (`LevelHandler.cpp:138-152`)
returns `{alreadyLoaded:true}` when the requested path matches the *current*
world, but a request for a *different* registered-but-unsaved world falls
straight through to the disk check and the misleading `FILE_NOT_FOUND`. The
in-memory-presence signal the handler already computes for the current world is
never extended to the not-current-but-loaded case.

## Producers of the orphan state (broader than the create_level case)

The `B-create-level-saved-true-no-umap` fix (in-tree at
`LevelStructureHandler.cpp:304-337`) tears down the in-memory world on the
`level.structure.create_level` save-failure path, so that *specific* repro is
narrowing. But the orphan state is reachable through other paths the message
defect still misdescribes:

- `level.create` (`LevelHandler.cpp:405-433`) has the **same un-fixed**
  stranding bug: it runs `GEditor->NewMap(false)` + `SetCurrentWorld(NewWorld)`,
  and when `FEditorFileUtils::SaveMap` fails it emits `SAVE_FAILED` (:427)
  **without** tearing the world down — leaving an unsaved in-memory `UWorld`.
- Any registered-but-unsaved world more generally (a world loaded in memory
  whose backing `.umap` is absent/deleted on disk) hits the identical blind
  spot. This is the same permanently-reachable two-state distinction the DONE
  `E-asset-dump-distinguish-file-missing-from-load-failed` shipped for assets.

So the message defect is independent of the create_level cause-fix and the
broader class has real producers — it is not made moot by that sibling.

## Fix (message clarity only — no behavior change required)

Before emitting `FILE_NOT_FOUND` at `LevelHandler.cpp:181-186` (and the alias's
own pre-delegation disk gate at `EditorCommandHandler.cpp:483-487` — the alias
runs `FPaths::FileExists` and emits its own "Level file not found" **before** it
delegates to `level.load` at :505, so fixing `level.load` alone does not cover
it), probe in-memory / registry state for the requested path:

1. `FindObject<UWorld>`/`FindPackage` (or an `IAssetRegistry` lookup) for the
   package — if a matching world/package exists in memory or the registry but
   no `.umap` is on disk, send a distinct, descriptive error
   (e.g. `LEVEL_NOT_PERSISTED` / `LEVEL_IN_MEMORY_ONLY`) whose message states
   the world is loaded/registered but was never saved to disk, so a disk load
   is impossible — and name the recovery (save it, or discard the orphan).
2. Keep `FILE_NOT_FOUND` for the genuinely-absent case (nothing in memory,
   registry, or disk), mirroring the `ASSET_FILE_MISSING` vs `ASSET_LOAD_FAILED`
   split that `E-asset-dump-distinguish-file-missing-from-load-failed` shipped.

Optional follow-on (out of scope here, behavior change): have `level.list`
flag registry entries with no backing `.umap` so the discovery surface itself
stops disagreeing with the load surface.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `level.structure`
  WP-map task. After `create_level` stranded an in-memory world with no `.umap`
  (the `B-create-level-saved-true-no-umap` bug), the agent hit
  `[FILE_NOT_FOUND] Level file not found: /Game/Maps/Highlands_WP` on every
  load attempt — while `level.list` simultaneously listed that exact path as an
  existing map. The contradictory, path-resolution-flavored error drove
  trial-and-error: 5 `level.load` variants (full / object-path / bare-name /
  two evict-target maps), 1 `editor.open_level`, 2 `editor.save_all` flushes
  (`savedCount:0 totalDirty:0`), and 2 `system.console.search` save-command
  probes — none recovered, per the friction note. Root cause of the unclear
  message: `LevelHandler.cpp:180-184` gates `FILE_NOT_FOUND` on disk presence
  only (`FileExists` + `DoesPackageExist`) and never checks in-memory/registry
  presence; alias `editor.open_level` emits the same at
  `EditorCommandHandler.cpp:476-477`. The already-loaded short-circuit at
  `LevelHandler.cpp:142-150` only covers the *current* world, not other
  registered-but-unsaved worlds. Distinct PROCESS/ergonomic angle from the
  judge-filed `B-create-level-saved-true-no-umap` (which fixes the orphan cause,
  not the load-path error wording). Direct sibling of the DONE
  `E-asset-dump-distinguish-file-missing-from-load-failed` (same disk-vs-present
  conflation, opposite blind spot, different namespace) — propose the analogous
  `FPackageName::DoesPackageExist` / in-memory pre-check + distinct
  `LEVEL_NOT_PERSISTED` code. Dedup: ripgrep + qmd across OPEN/closed found no
  existing ticket on the `level.load` FILE_NOT_FOUND-vs-in-memory error wording.
- `#2-reword` `OPEN` developer — Corrected stale line citations against current
  source and broadened the producer analysis before implementing. The
  `level.load` emit is `LevelHandler.cpp:181-186` (was `180-184`); the
  already-loaded short-circuit is `:138-152` (was `142-150`). The alias citation
  `EditorCommandHandler.cpp:476-477` was wrong — that line is a *different*
  FILE_NOT_FOUND ("Level path has no registered mount point"); the alias's "Level
  file not found" emit is its own pre-delegation disk gate at
  `EditorCommandHandler.cpp:483-487`, which fires BEFORE it cross-dispatches to
  `level.load` at :505, so the fix must touch both surfaces. Added a "Producers"
  section: the create_level orphan is being torn down by the in-tree
  `B-create-level-saved-true-no-umap` fix (`LevelStructureHandler.cpp:304-337`),
  but `level.create` (`LevelHandler.cpp:405-433`) still strands an unsaved
  in-memory `UWorld` on `SaveMap` failure (emits `SAVE_FAILED` with no teardown),
  and any registered-but-unsaved world hits the same blind spot — so the message
  defect is not made moot by the cause-fix and the broader class has real
  producers. (Adversarial "no remaining producer" claim missed `level.create`.)
- `#3-fix` `IN-REVIEW` developer — Split the disk-only FILE_NOT_FOUND verdict so
  an in-memory/registry-present-but-unsaved world gets a distinct, actionable
  `LEVEL_NOT_PERSISTED` instead of the misleading `FILE_NOT_FOUND`. Added a pure,
  unit-testable classifier `ClassifyLevelLoadability(bFileOnDisk,
  bInMemoryOrRegistryPresent)` in `Utils/AssetUtils.{h,cpp}` returning
  `ELevelLoadability::{Loadable, NotPersisted, Missing}` (mirrors the existing
  `ShouldTreat*SaveAsSuccess` decision-helper seam). `LevelHandler.cpp:181-186`
  now computes the in-memory/registry signal (`FindObject<UWorld>` /
  `FindPackage` + an asset-registry `World`-class probe) and, when the file is
  absent but the world is present in memory/registry, emits `LEVEL_NOT_PERSISTED`
  ("registered/loaded in memory but never saved to disk — save it or discard the
  orphan; it cannot be loaded from disk"); the genuinely-absent case keeps
  `FILE_NOT_FOUND`. The alias `editor.open_level` got the same split at its own
  pre-delegation disk gate (`EditorCommandHandler.cpp:483-487`) so it no longer
  short-circuits an in-memory world with a flat FILE_NOT_FOUND before delegating.
  Added `ERR_LEVEL_NOT_PERSISTED` to `Handlers/ErrorCodes.h`. Files:
  `Private/Utils/AssetUtils.h`, `Private/Utils/AssetUtils.cpp`,
  `Private/Handlers/Level/LevelHandler.cpp`,
  `Private/Handlers/Editor/EditorCommandHandler.cpp`,
  `Private/Handlers/ErrorCodes.h`. Test:
  `Private/Tests/Core/TestLevelSaveLoadUtils.cpp` —
  `...should_classify_level_loadability.*` asserts the three-way verdict
  (disk→Loadable, in-memory/registry-only→NotPersisted, nothing→Missing); it
  fails if the classifier collapses back to a disk-only FILE_NOT_FOUND.
