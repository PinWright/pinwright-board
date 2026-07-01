---
id: E-level-structure-wp-wiki-advertises-dead-end
title: "level.structure wiki routes the World Partition flow through enable_world_partition (which refuses on a non-WP active world) and never mentions the working create_level {bCreateWorldPartition:true} path"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, level-structure, world-partition, enable, create-level, discoverability, wiki]
---

# The `level.structure` wiki points at the refusing enable path and hides the working create path

`docs/wiki-src/level.structure.md` presents the World-Partition authoring flow
as a single canonical sequence that begins with `enable_world_partition` — but
that verb **refuses on an already-loaded non-WP world**, and the page never
mentions the one path that actually does enable World Partition through the MCP:
`create_level {bCreateWorldPartition:true}`.

- Line 7 (Cross-cluster overlap): *"The typical flow is `call("level.create", …)`
  first, then `call("level.structure.enable_world_partition", …)` and the
  data-layer / HLOD / streaming setup."* — `level.create` produces a plain
  (non-WP) world, so step two (`enable_world_partition`) then hard-refuses; the
  documented "typical flow" cannot complete.
- Lines 15-27 (the `enable_world_partition` section): a full 5-step workflow —
  snapshot → **enable** → tune cell size → define data layers → assign actors —
  described as *"destructive to the legacy sublevel composition"* and
  *"irreversible without restoring a backup."* Step 2 is the `enable_world_partition`
  call that refuses on a non-WP active world, so the workflow as written stalls
  at step 2.

## What actually works now (post-`F-enable-world-partition-impossible`)

`F-enable-world-partition-impossible` shipped in commit `12aa22e` ("GO …
[tests:CLEAN]"). As of that commit, `create_level {bCreateWorldPartition:true}`
attaches a **real** `UWorldPartition`:

- `LevelStructureHandler.cpp:238-247` builds
  `UWorld::InitializationValues().CreateWorldPartition(bCreateWorldPartition)`
  and calls `UWorld::CreateWorld(WorldType, …, &IVS)`.
- `:267-271` derives `bWorldPartitionActuallyEnabled = (NewWorld->GetWorldPartition() != nullptr)`
  — an honest readback, not the old hardcoded `false`.
- `:344` reports that real state as `worldPartitionEnabled`.

On a world created that way, `World->GetWorldPartition()` is non-null, so the
whole WP-gated namespace (`configure_grid_size`, `create_data_layer`,
`configure_hlod_layer`, `configure_level_bounds`, `create_minimap_volume`,
`assign_actor_to_data_layer`) is **reachable** — the create-with-flag path is the
supported route to a partitioned world.

What still does **not** work is converting an already-loaded non-WP world:
`enable_world_partition {bEnableWorldPartition:true}` on such a world hard-refuses
`[OPERATION_FAILED] Cannot enable World Partition programmatically. Use 'Edit >
Convert Level' in editor or create a new level with World Partition enabled.`
(`LevelStructureHandler.cpp:786-792`). The F- ticket's history deliberately left
the in-place conversion unchanged.

## The residual doc gap (narrow)

The wiki sequences the flow backwards from current reality: it leads with the
refusing `enable_world_partition` and omits the working
`create_level {bCreateWorldPartition:true}` path entirely. A caller following the
"typical flow" creates a plain level, calls `enable_world_partition`, and hits an
`[OPERATION_FAILED]` with no hint that the supported route is to set the WP flag
at create time instead. That is a real discoverability miss on a core
open-world-setup task — but it is narrow (one cross-ref + one caveat), not the
sweeping "every WP path is a dead end" the original audit claimed.

## Why this is a distinct PROCESS angle (not a re-file of the F- ticket)

`F-enable-world-partition-impossible` is the **capability** ticket (now IN-REVIEW
after `12aa22e`). It quotes the wiki as evidence but its action is a pure C++
change; it does not propose the wiki-overlay fix. This ticket is the
**docs-overlay process fix**: re-point the page at the now-working create-with-flag
path and caveat the `enable_world_partition` in-place refusal. Same auditor/judge
docs-companion shape as the accepted precedents
`E-ik-rig-family-wiki-advertises-compiled-out-workflow` and
`E-set-transition-rules-wiki-overstates-rule-authoring`.

It is also orthogonal to the other `level.structure`/WP tickets:
`B-create-level-saved-true-no-umap` (false `saved:true` / no-`.umap` persistence),
`B-level-create-makes-wp-map` (`level.create`'s `NewMap(true)` contradiction),
`B-configure-world-partition-silent-noop` / `E-perf-wp-configure-readback-thin`
(`performance.configure_world_partition` CVar echo). None of those touch the
`level.structure` wiki page.

## Fix (wiki overlay — `docs/wiki-src/level.structure.md`)

The generated `wiki-generated/level.structure.enable_world_partition.md`
regenerates from this source:

1. Line 7 (Cross-cluster overlap "typical flow"): replace the
   `level.create` → `enable_world_partition` sequence with the **working** route —
   create the level with World Partition already enabled
   (`call("level.structure.create_level", { bCreateWorldPartition: true })`),
   which attaches a real `UWorldPartition` and unlocks the data-layer / HLOD /
   streaming surface on that world. Do **not** state that "no programmatic WP path
   exists" — that is false post-`12aa22e`.
2. Lines 15-27 (the `enable_world_partition` section): keep the section but add a
   caveat that `enable_world_partition` **cannot convert an already-loaded non-WP
   world** — it returns `[OPERATION_FAILED]` ("Use Edit > Convert Level in editor,
   or create a new level with World Partition enabled"). Re-point the 5-step
   workflow's first step at `create_level {bCreateWorldPartition:true}` instead of
   create-then-enable. Cross-reference `F-enable-world-partition-impossible` for the
   future in-place-conversion path it deferred.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `level.structure`
  "OpenWorldProto" World-Partition-map task (15 calls; judge filed
  `F-enable-world-partition-impossible` for the capability/code root cause).
  PROCESS finding: `docs/wiki-src/level.structure.md` (line 7 "typical flow" + the
  lines 15-27 `enable_world_partition` 5-step workflow) advertise the WP
  enable/author flow with no caveat that the enable step refuses on a non-WP world.
  [NOTE: the initial audit also claimed `create_level {bCreateWorldPartition:true}`
  was a no-op stub at `:240-247` and that the whole WP surface was unreachable — that
  premise was invalidated by the `12aa22e` create-with-flag fix; see `#2-reword`.]
- `#2-reword` `OPEN` developer — REWORDED to current source reality. The original
  "both paths dead-end / create is a no-op stub / whole WP surface unreachable"
  premise is stale: `F-enable-world-partition-impossible` shipped in `12aa22e`
  ("GO … [tests:CLEAN]"), so `create_level {bCreateWorldPartition:true}` now attaches
  a real `UWorldPartition` (`LevelStructureHandler.cpp:238-247`, honest readback
  `:267-271`, reported `:344`) and the WP-gated namespace IS reachable on a
  level created that way. The original Fix would have written a FALSE "no programmatic
  WP path exists" claim into the wiki. Re-scoped to the surviving narrow gap: the wiki
  still leads with the refusing `enable_world_partition` (still refuses on a non-WP
  active world, `LevelStructureHandler.cpp:786-792`) and never mentions the working
  `create_level {bCreateWorldPartition:true}` route. Title/body/Fix/severity rewritten;
  severity Medium → Low (the page misdirects but no longer advertises a wholly
  non-functional path).
- `#3-reword-fix-applied` `IN-REVIEW` developer — Applied the reworded wiki-overlay
  fix to `Docs/wiki-src/level.structure.md` (the only file changed). (1) Cross-cluster
  overlap "typical flow": re-pointed from `level.create` then `enable_world_partition`
  (which refuses) to the working `create_level {bCreateWorldPartition:true}` route,
  and added a Gotcha that `enable_world_partition` cannot convert an already-loaded
  non-WP world (returns `[OPERATION_FAILED]`), cross-ref `F-enable-world-partition-impossible`.
  (2) `### level.structure.enable_world_partition` section: rewrote the lead to state it
  reports state / cannot enable in-place, and re-pointed the workflow's first step
  from snapshot-then-enable to `create_level {bCreateWorldPartition:true}` (dropped the
  snapshot-before-destructive-enable safety theater for a call that refuses). No C++
  changed; no regression test — wiki-overlay prose has no automation-test harness (same
  shape as the accepted `E-ik-rig-family-wiki-advertises-compiled-out-workflow` /
  `E-set-transition-rules-wiki-overstates-rule-authoring` docs-overlay fixes).
  Verified against current source: `LevelStructureHandler.cpp:238-247,267-271,344`
  (create attaches real WP) and `:786-792` (enable still refuses).
