---
id: E-orphan-world-no-discard-verb
title: "LEVEL_ALREADY_EXISTS for an unsaved in-memory world says \"Use load_level\", which cannot load it, and no verb discards the orphan"
status: OPEN
severity: Low
category: ergonomic
tags: [in-memory-orphan-world, level, create, orphan, discard, gc]
encounters: 1
lastSeen: 2026-07-04T16:30:08.0157490+03:00
rice: [1, 2, 0.8, 2]
priority: 7
---

# An unsaved in-memory world blocks re-creation, and the recovery hint points at a verb that cannot work

Unsaved worlds still get left in memory at a `/Game/` path: `level.structure.create_level
{save:false}`, `level.duplicate` (in memory only, `docs/wiki-src/level.md:193-197`), and a
`level.create` whose save fails (`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:743-749`
sends the error with no teardown). `create_level`'s own save-failure path does tear its world down
(`LevelStructureHandler.cpp:420-432`).

The next `level.structure.create_level` at that path hits the in-memory guard
(`LevelStructureHandler.cpp:296-306`):

> `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/LightingSandbox. Use load_level or provide a different name.`

Both suggestions fail the caller:

- `level.load` on an unsaved path fails `LEVEL_NOT_PERSISTED` (nothing on disk).
  `docs/wiki-src/level.md:208` then says "save it or discard the orphan", but no verb discards one.
- A different name only sidesteps the collision and can leave another orphan.

The working escape is undocumented: `level.load` an unrelated on-disk map so the orphan is
garbage-collected. In the audited task the agent did this twice (for `/Game/Maps/LightingSandbox` and
`/Game/Maps/LS_Tmp`), side-loading `/Game/Maps/ExampleProjectWelcome` each time.

**Workaround:** if the orphan is the active world, save it with `level.save` / `level.save_as`.
Otherwise `level.load` any real map to let GC reclaim it, then retry.

**Fix:**
1. Make the in-memory `LEVEL_ALREADY_EXISTS` message state that the world was never saved and name
   the real choices: save it (`level.save` / `level.save_as`), discard it, or use another name. Drop
   "Use load_level" from that branch.
2. Add a discard path for an unsaved in-memory world (a `level.discard` verb, or a flag on
   `create_level` that tears down an existing unsaved world at the path). If no verb is added,
   document the load-another-map escape in `level.md` where it says "discard the orphan".

**Acceptance:** after `level.structure.create_level {name:X, save:false}`, a second
`create_level {name:X}` either succeeds after the documented discard step or returns a message that
names a step which works. Following that step makes the third call succeed without loading an
unrelated map.

Related: `E-level-load-file-not-found-vs-in-memory-orphan` (the `LEVEL_NOT_PERSISTED` wording),
`B-create-level-saved-true-no-umap` (teardown on `create_level`'s own failed save),
`E-create-level-save-false-world-not-active` (activation verb, out of scope).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a lighting-sandbox setup task (create /Game/Maps/LightingSandbox with a directional + point light, save, back up, delete backup). The run was blocked by the level-persistence cluster (filed/updated under `B-create-level-saved-true-no-umap` `#5`, `B-lighting-create-level-false-success-no-umap`, `B-level-save-saved-true-in-memory-no-umap`), but a distinct PROCESS cost surfaced on top: `level.create` left an unpersistable in-memory phantom at `/Game/Maps/LightingSandbox`, so the follow-up `level.structure.create_level {LightingSandbox, save:true}` hit `[LEVEL_ALREADY_EXISTS] Level already exists in memory: /Game/Maps/LightingSandbox. Use load_level or provide a different name.` — but `load_level` on that path fails (nothing on disk) and no discard verb exists, so the agent side-loaded `/Game/Maps/ExampleProjectWelcome` purely to GC the orphan, then repeated the identical eviction for a second `LS_Tmp` phantom. Two decoy-map loads spent on cleanup that a discard/reset verb (or an accurate collision message) would eliminate. Dedup: distinct from `B-create-level-saved-true-no-umap` (tears down only create_level's own orphan, not a level.create phantom), `B-level-create-makes-wp-map` (what world create builds), `E-level-load-file-not-found-vs-in-memory-orphan` (load error wording; its `#2` confirms level.create still strands the orphan but files no discard verb), and `E-create-level-save-false-world-not-active` (activation verb, out of scope). Ripgrep over OPEN/closed for discard/unload/reset-world/set-active/evict found no ticket on the missing discard verb or the load-a-decoy eviction workaround.
- `#2-rephrased` `OPEN` developer — Dropped the failed-save framing: the wrong-drive save that stranded most orphans is fixed (`LevelHandler.cpp:721-725` saves to the resolved filename), and `create_level` tears down its own failed save (`LevelStructureHandler.cpp:420-432`). Orphans now come mainly from `create_level {save:false}`, `level.duplicate`, and a failed `level.create` save, which still has no teardown (`LevelHandler.cpp:743-749`). Still real: the in-memory `LEVEL_ALREADY_EXISTS` hint says "Use load_level" (`LevelStructureHandler.cpp:303-304`), `level.md:208` says "discard the orphan" with no verb to do it, and there is no discard verb. Fix restated as a correct hint plus a discard path, or a documented escape. Added Workaround and Acceptance. Severity unchanged (Low).
