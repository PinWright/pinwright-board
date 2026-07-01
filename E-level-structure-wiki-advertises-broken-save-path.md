---
id: E-level-structure-wiki-advertises-broken-save-path
title: "level.structure wiki tells callers to persist the in-memory WP world with {save:true} (a live SAVE_VERIFICATION_FAILED dead-end) and never warns that NO MCP save verb persists an in-memory WP world today, driving a repeated save-retry loop + C++ source dive"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, level-structure, world-partition, create-level, save, save-as, in-memory, persistence, discoverability, wiki, retry-loop]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# The `level.structure` wiki advertises a save path that is a live dead-end

`docs/wiki-src/level.structure.md` line 26 (the `### level.structure.enable_world_partition`
workflow section) tells the caller how to persist a `{save:false}` in-memory
World-Partition world:

> "`create_level` `{save:false}` builds the world in memory only — it is reachable
> now (it is the active editor world), but `call("level.load", …)` will refuse the
> path with `[LEVEL_NOT_PERSISTED]` until it is saved to disk. **Use `{save:true}`
> (the default) to also write the `.umap`.**"

That last sentence is a **live dead-end**. As of the current build, `{save:true}`
returns `[SAVE_VERIFICATION_FAILED] Level created but no .umap was written to disk`
and tears the world down (the persistence defect tracked in
`B-create-level-saved-true-no-umap`, IN-REVIEW); `level.save` and `level.save_as`
on the active in-memory WP world *also* return `SAVE_VERIFICATION_FAILED`
(honestly `saved:false` now, post-`B-level-save-saved-true-in-memory-no-umap` `#2`).
So **no MCP save verb persists an in-memory WP world today**, yet the wiki points
the caller straight at `{save:true}` as the remedy and gives no warning that the
whole save surface is currently a dead-end for this workflow.

The result is a caller who follows the documented workflow — `create_level
{bCreateWorldPartition:true}` → (build data layers + actors) → "save the map" —
has **no working save path** and **no doc hint that there isn't one**, so the
only way to learn that is to exhaust every save verb and then read plugin C++.

## Why this is a distinct PROCESS angle (not a re-file of the persistence bugs)

The OUTCOME (the missing-`.umap` persistence defect) is fully owned by the
already-filed tool bugs, and `B-create-level-saved-true-no-umap` `#4` already
cites THIS exact `OpenWorldSandbox` cold-load task verbatim. Those are **C++
fixes**; none of them touches this wiki line or the missing "stop retrying" caveat:

- `B-create-level-saved-true-no-umap` (IN-REVIEW, High) — the `create_level`
  `{save:true}` disk-write defect + teardown. Code fix in `LevelStructureHandler.cpp`
  / `AssetUtils.cpp`; it does not edit `docs/wiki-src/level.structure.md` line 26 or
  add a "no save verb works for an in-memory WP world" note.
- `B-level-save-saved-true-in-memory-no-umap` (IN-REVIEW, High) — made `level.save`/
  `level.save_as` report honest `saved:false` for an in-memory world. Its `#1`
  captured the *false-success* process friction; with that now fixed, the
  **residual** friction is that the wiki still advertises `{save:true}` as the cure
  and the caller still has nowhere to land the file — a docs gap, not a false-success.
- `E-level-structure-wp-wiki-advertises-dead-end` (IN-REVIEW) — the *adjacent* wiki
  fix on the SAME page, but a different sentence: it re-pointed the page from
  `enable_world_partition` to `create_level {bCreateWorldPartition:true}`. It made
  the *enable-vs-create* misdirection right; it did NOT touch line 26's `{save:true}`
  persistence advice, which is still a dead-end.
- `E-create-level-save-false-world-not-active` (IN-REVIEW) — the `{save:false}`
  inactive-world gap (now fixed: the world IS active, which is exactly why this task
  got far enough to build all 3 data layers + 3 actors in memory before hitting the
  unfixable save). Not the save-path-misadvertised angle.

So this is the narrow **docs-overlay ergonomic** companion: re-word line 26 so it
no longer prescribes the broken `{save:true}` cure and instead states that an
in-memory WP world cannot currently be persisted to disk via any MCP verb
(cross-ref the B- persistence tickets), so the caller stops at the first failure
instead of looping. Same auditor/judge docs-companion shape as the accepted
`E-level-structure-wp-wiki-advertises-dead-end` on this very page.

## Evidence (from the audited task — `level.structure.create_data_layer`, OpenWorldSandbox WP-map setup, 41 calls)

Outcome: the data-layer / actor-assignment core all succeeded in memory (3 data
layers correct types+visibility, 2 StaticMeshActors→Foliage + 1 PointLight→Enemies,
structure readback + `actor.describe DataLayerAssets` confirmed). The ONLY failure
was the explicit "save the map" step — and the friction is that the wiki sent the
caller into a dead-end with no stop signal. The call-log shows the cost as a
**doubled** save-retry loop bracketing the in-memory work:

- First loop (right after `create_level {save:false}` made the world active):
  `level.save` → `SAVE_VERIFICATION_FAILED` → `system.job_status` poll →
  `level.save_as` → `SAVE_VERIFICATION_FAILED` → `system.job_status` poll.
- Then the whole scene was built in memory (3× `create_data_layer`, 2× `actor.spawn`,
  `spawn_light`, 3× `assign_actor_to_data_layer`, readbacks).
- Second loop (the final "save the map" step): `level.save` →
  `SAVE_VERIFICATION_FAILED` → `system.job_status` poll → `level.save_as` →
  `SAVE_VERIFICATION_FAILED` → `system.job_status` poll — the **identical**
  dead-end, re-run because nothing told the caller the first loop's verdict was
  permanent.

Friction note, verbatim:

> "Struggled hard on persistence. The wiki's documented WP workflow (create_level
> {bCreateWorldPartition:true, save:true}) is broken … Worked around by using
> save:false (which does make it active in memory), then tried level.save and
> level.save_as — both also fail. Had to read plugin C++ (LevelStructureHandler.cpp,
> AssetUtils.cpp) and the editor log as a last resort to diagnose … No MCP path
> persists an in-memory WP world to disk."

That C++/log source dive is the discoverability cost the wiki caveat would remove:
a single line stating "no MCP save verb currently persists an in-memory WP world"
turns ~6 wasted save+poll calls and a source dive into one informed stop.

## Fix (wiki overlay — `docs/wiki-src/level.structure.md`)

Re-word line 26 (the `create_level {save:false}` persistence note) so it no longer
prescribes `{save:true}` as the cure. State plainly that an in-memory World-
Partition world created this way **cannot currently be persisted to disk via any
MCP verb** — `{save:true}` returns `SAVE_VERIFICATION_FAILED` and tears the world
down (`B-create-level-saved-true-no-umap`), and `level.save` / `level.save_as` on
the active in-memory WP world likewise return `SAVE_VERIFICATION_FAILED`
(`B-level-save-saved-true-in-memory-no-umap`). So the caller should treat the first
`SAVE_VERIFICATION_FAILED` as terminal for this workflow (do not loop the save verbs)
and track the persistence work via those tickets, rather than discovering the
dead-end by exhausting every save path and reading plugin C++. Keep the existing
`LEVEL_NOT_PERSISTED` / `level.load` note. When the B- persistence fix lands, update
this caveat to the now-working path in the same pass.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `level.structure.create_data_layer` "OpenWorldSandbox" open-world WP-map setup task
  (41 calls; the per-finding judge filed/updated `B-create-level-saved-true-no-umap`
  `#4`, which cites this exact cold-load task, for the persistence tool bug). PROCESS
  finding: `docs/wiki-src/level.structure.md` line 26 advertises `{save:true}` as the
  way to persist a `{save:false}` in-memory WP world, but `{save:true}`,
  `level.save`, and `level.save_as` all return `SAVE_VERIFICATION_FAILED` on an
  in-memory WP world today, and the page gives no "no MCP save verb persists an
  in-memory WP world" caveat — so the agent ran the `level.save → poll → level.save_as
  → poll` dead-end loop TWICE (once after `create_level {save:false}`, once as the
  final save step) and then fell back to reading `LevelStructureHandler.cpp` /
  `AssetUtils.cpp` + the editor log to diagnose (friction note verbatim above).
  Proposed remedy: re-word line 26 to state the in-memory WP world cannot be persisted
  via any MCP verb today, cross-ref `B-create-level-saved-true-no-umap` /
  `B-level-save-saved-true-in-memory-no-umap`, so the first `SAVE_VERIFICATION_FAILED`
  is treated as terminal instead of looped. Dedup: ripgrep over OPEN + closed
  (`SAVE_VERIFICATION_FAILED`, `save:true`, `in-memory`, `level.structure`,
  `world-partition`, `save.*dead.end`) — distinct from `B-create-level-saved-true-no-umap`
  / `B-level-save-saved-true-in-memory-no-umap` (the C++ persistence defects; neither
  edits this wiki line nor adds the stop-retrying caveat), from
  `E-level-structure-wp-wiki-advertises-dead-end` (same page but the enable-vs-create
  sentence, line 7/section 15-27, not line 26's save advice), and from
  `E-create-level-save-false-world-not-active` (the `{save:false}` inactive-world gap,
  now fixed). No existing ticket targets line 26's misadvertised `{save:true}` save
  path or the missing "no MCP save verb persists an in-memory WP world" caveat.
