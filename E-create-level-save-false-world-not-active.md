---
id: E-create-level-save-false-world-not-active
title: "level.structure.create_level never makes the created world the active editor world, so the WP-gated level.structure verbs can't target it"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level-structure, create-level, save-false, active-world, in-memory, world-partition, no-win-fork, discoverability]
---

# `create_level {save:false}` leaves the new world inactive and unreachable

`level.structure.create_level` has two persistence modes and **both are dead
ends for the WP-authoring workflow** the namespace exists to serve:

- `save:true` (the default) → `[SAVE_VERIFICATION_FAILED]` (no `.umap`) and the
  handler tears the world down. That persistence defect is tracked in
  `B-create-level-saved-true-no-umap` (IN-REVIEW; its `#3` already records this
  exact seed re-hitting it for both WP and flat).
- `save:false` → the call **succeeds** and builds a live world in memory (post
  `F-enable-world-partition-impossible`'s `12aa22e`, with a real `UWorldPartition`
  attached, so it honestly reports `worldPartitionEnabled:true`) — **but the
  created world is never made the active editor world.** Every WP-gated
  `level.structure` verb (`configure_grid_size`, `create_data_layer`,
  `configure_minimap_volume` / `create_minimap_volume`, `configure_hlod_layer`,
  `configure_level_bounds`, `assign_actor_to_data_layer`,
  `get_level_structure_info`) resolves its target through the **active editor
  world** (`GetEditorWorldLS` in `LevelStructureHandler.cpp`), so none of them can
  see the freshly-created-but-inactive world. They operate on whatever map was
  already active (`ExampleProjectWelcome`).

So the WP scaffold (grid / data layer / minimap volume) the caller created the
level *for* can never be reached, and **there is no MCP verb to promote the
in-memory world to active**:

- `level.load {/Game/Maps/<name>}` → `[LEVEL_NOT_PERSISTED]` ("registered/loaded
  in memory but never saved to disk … do not retry path forms") — it refuses
  precisely because `save:false` wrote no `.umap`.
- `level.save {}` / `level.save_as {}` target the *active* world
  (`ExampleProjectWelcome`), not the orphaned new world, so they cannot persist
  it into reach either.

The two modes form a **no-win fork**: `save:true` fails to persist and destroys
the world; `save:false` succeeds but produces a live world that no subsequent
verb can target or activate. Even with the B- persistence fix and the F- WP-attach
fix both landed, the `save:false` branch still strands the world — the missing
piece is "make the created world the active editor world (or expose a verb that
does)."

## Why this is a distinct (process) angle

- `B-create-level-saved-true-no-umap` (IN-REVIEW) is the `save:true` persistence
  defect (false `saved:true` → now `SAVE_VERIFICATION_FAILED` + teardown). It does
  not address what happens on the `save:false` branch.
- `F-enable-world-partition-impossible` (IN-REVIEW) made `create_level
  {bCreateWorldPartition:true}` attach a real `UWorldPartition` — but its
  regression test runs the handler against a throwaway GUID path with `save:false`
  and asserts only the *response field* + `IsPartitionedWorld()` on the returned
  world; it never exercises a *follow-on* WP verb, so the "created world is not the
  active world the WP verbs read" gap is outside its scope and untested.
- `E-level-create-active-world-mismatch` (IN-REVIEW) is the **opposite** behavior
  on the **legacy `level.create`** verb: that handler runs
  `GEditor->NewMap(false)` + `SetCurrentWorld(NewWorld)` so it *does* leave the
  created world active (its `#2` confirms the active world == the `/Game/` package
  after the rename). `level.structure.create_level` does **not** call
  `SetCurrentWorld`/`SetCurrentWorld(NewWorld)` on the `save:false` path, so the
  created world is left inactive. Different verb, opposite symptom.
- `E-level-load-file-not-found-vs-in-memory-orphan` (IN-REVIEW) fixed the
  `level.load` *wording* (the `LEVEL_NOT_PERSISTED` message this task saw) — it
  explains why `level.load` refuses, but offers no route to activate the in-memory
  world; the gap here is the missing activation path, not the message.

## Fix

**Make the created world active** (the smallest change, mirroring the pattern
`level.create` already uses). On both the `save:false` branch and the post-
successful-`save:true` path — i.e. wherever the handler falls through to
`SendSuccess` — promote the new world to `EWorldType::Editor` and call
`GEditor->GetEditorWorldContext().SetCurrentWorld(NewWorld)`, so the follow-on WP
verbs (`configure_grid_size` / `create_data_layer` / `create_minimap_volume` / …),
which all resolve through the active editor world (`GetEditorWorldLS`), target the
world the caller just created. That makes `create_level → configure_grid_size →
create_data_layer → create_minimap_volume` a coherent sequence.

**Also disclose it.** The `create_level` response should carry an `activeWorld`
boolean and a `note` confirming the world is now the active editor world (or, in a
no-`GEditor` context, that it could not be made active), plus a `persistenceNote`
that `save:false` writes no `.umap`, so `level.load` will refuse the path with
`LEVEL_NOT_PERSISTED` until it is saved.

A dedicated activation verb (e.g. `level.set_active` / a `makeActive:true` flag) is
out of scope — activating on creation removes the need for one.

### Coordinate with `B-create-level-saved-true-no-umap`

`B-`'s `#3` already prescribes "set the created world active before SaveLevel" as
the remaining fix for its `save:true` disk-write defect. That overlaps the
`SetCurrentWorld` change here. They are not duplicates — different branch
(`save:true` vs `save:false`), different symptom (no `.umap` vs WP-verb-unreachable)
— but the implementer must ensure a single activation change isn't done twice or in
conflict across the two tickets.

## Workaround (none clean before this fix)

Before this fix there was **no** working MCP sequence: `save:true` failed to persist
(B-), and `save:false` left the world inactive with no verb to activate it. After
the fix the created world is active, so the WP verbs reach it directly; persistence
of a `save:false` world still requires `save:true`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the
  `level.structure.create_minimap_volume` task (create a WP "Frontier_WP" map, set
  grid 25600/51200, add a "Landscape" data layer, place a "FrontierMinimap" volume,
  then summarize). 14 calls, hard dead-end, zero downstream WP steps reachable. The
  no-win fork showed directly: `create_level {Frontier_WP, WP, save:true}` →
  `[SAVE_VERIFICATION_FAILED] … no .umap written … /Game/Maps/Frontier_WP` then the
  world is torn down; `create_level {Frontier_WP, WP, save:false}` → `ok` (live WP
  world in memory) but never set active; `level.load {/Game/Maps/Frontier_WP}` →
  `[LEVEL_NOT_PERSISTED]`; `level.save {}` saved the *active* `ExampleProjectWelcome`
  (wrong world); retry `create_level {save:true}` → `[LEVEL_ALREADY_EXISTS] … in
  memory: /Game/Maps/Frontier_WP`; fresh name `Frontier_WP2 {save:true}` → the same
  `[SAVE_VERIFICATION_FAILED]`. Friction note verbatim: "create_level{save:false}
  builds the WP world in memory but never makes it the active editor world, so the
  GEditor active-world-based WP handlers (configure_grid_size/create_data_layer/
  create_minimap_volume/get_level_structure_info) can't target it; level.load
  refuses (LEVEL_NOT_PERSISTED) and level.save/save_as only hit the active
  ExampleProjectWelcome." The auditee had to read `LevelStructureHandler.cpp` as a
  last resort to confirm the `GetEditorWorldLS` active-world resolution and the
  save-then-destroy logic — itself a discoverability gap. Distinct PROCESS angle
  from the judge-filed `B-create-level-saved-true-no-umap` (the `save:true`
  persistence defect — already carries this seed at its `#3`), from
  `F-enable-world-partition-impossible` (WP-attach capability, regression test
  never exercises a follow-on WP verb), from `E-level-create-active-world-mismatch`
  (legacy `level.create`, which *does* `SetCurrentWorld` the created world — opposite
  symptom, different verb), and from `E-level-load-file-not-found-vs-in-memory-orphan`
  (the `LEVEL_NOT_PERSISTED` *wording*, not the missing activation path). Dedup:
  ripgrep over OPEN + closed (`create_level`, `set.active`, `GetEditorWorldLS`,
  `save:false`, `inactive.world`) found no ticket on the `save:false` created world
  being left inactive / unreachable by the WP-gated verbs nor on the absence of an
  activation verb for a registered-but-unsaved in-memory world.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: narrowed the three-option
  remedy to the root-cause fix (activate the created world, mirroring `level.create`)
  plus a disclosure note; dropped the new-activation-verb option (gold-plating — out
  of scope once the world is activated on creation); added a coordination note for
  `B-create-level-saved-true-no-umap` `#3` (its `save:true` "set the created world
  active before SaveLevel" change overlaps this `SetCurrentWorld`). Implemented in
  `LevelStructureHandler.cpp` `level.structure.create_level`: after creation (on the
  `save:false` branch and after a successful `save:true` save — a save FAILURE still
  returns early with `SAVE_VERIFICATION_FAILED` + teardown), the handler now sets
  `NewWorld->WorldType = EWorldType::Editor` and calls
  `GEditor->GetEditorWorldContext().SetCurrentWorld(NewWorld)`, so the active-world-
  resolving WP verbs (`configure_grid_size` / `create_data_layer` /
  `create_minimap_volume` / `get_level_structure_info` / …) now target the created
  world. The response gained an `activeWorld` boolean, a `note` (active-world
  confirmation, or no-`GEditor` disclosure), and a `persistenceNote` (`save:false`
  writes no `.umap`; `level.load` refuses `LEVEL_NOT_PERSISTED`). Wiki overlay
  `docs/wiki-src/level.structure.md` workflow step 1 updated to state the created
  world becomes the active world (`activeWorld:true`) and the `save:false`/`level.load`
  persistence constraint. Regression test:
  `EditorAutomationRpcGateway.level.structure.create_level.MakesWorldActive` in
  `Tests/World/TestLevelHandlers.cpp` invokes the production handler with `save:false`
  and asserts (1) the response reports `activeWorld:true` and (2) ground truth —
  `GEditor->GetEditorWorldContext().World()` IS the created world and differs from the
  pre-call active world; reverting `SetCurrentWorld` fails both. The existing
  `F-`-regression test `…create_level.AttachesWorldPartition` was updated to wrap a
  `FScopedEditorWorldMapGuard` (the activation now swaps the active world, which must
  be restored for the rest of the suite).
