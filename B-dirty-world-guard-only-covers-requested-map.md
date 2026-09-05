---
id: B-dirty-world-guard-only-covers-requested-map
title: "`level.load`'s DIRTY_WORLD_BLOCKS_MAP_SWAP guards only the requested map, while the fatal it prevents is triggered by the outgoing resident world"
status: OPEN
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
history on `B-level-load-dirty-world-leak-fatal` (`#2`, filed from this project): the swap that
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

Distinct from `B-level-load-dirty-world-leak-fatal`, which reports the crash itself and whose fix
added the requested-map guard. This ticket is about the half of the mechanism that guard does not
cover. Fixing it should reuse the same refusal path and payload shape.
