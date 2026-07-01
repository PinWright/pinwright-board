---
id: E-world-partition-wiki-omits-active-map-precondition
title: "world_partition namespace wiki never states the active-map-must-be-WP precondition, so the natural first call (load_cells on the default non-WP startup map) fails [NOT_PARTITIONED] before any work begins"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, world-partition, load-cells, precondition, discoverability, wiki, not-partitioned]
encounters: 2
lastSeen: 2026-06-24T09:27:12Z
---

# The `world_partition` wiki omits the "active map must be partitioned" precondition

Every verb in the `world_partition` namespace (`load_cells`, `create_datalayer`,
`set_datalayer`, `cleanup_invalid_datalayers`, the HLOD verbs) operates on the
**active editor world** and silently assumes that world is World-Partitioned.
But the editor's default startup / initial map is **not** partitioned, so the
first natural call against the active world fails its precondition with
`[NOT_PARTITIONED] World is not partitioned.` — and the namespace wiki gives the
caller no warning and no remedy.

`docs/wiki-src/world_partition.md` is a **one-line stub** — a single description
paragraph with no preconditions section. It tells you what the namespace is for
("cell loading, Data Layer creation/assignment, cleanup…") but never says:

1. these verbs require the **active world to be a World Partition map**, and
2. the editor's initial/default active map typically is **not** partitioned, so
   you must `editor.open_level <a WP map>` (or create one with
   `level.structure.create_level {bCreateWorldPartition:true}`) **first**, and
3. how to confirm WP state up front (e.g. `level.structure.get_level_structure_info`
   reports `worldPartitionEnabled`) so you don't discover it via a failed verb.

A level designer's natural opening move — "load the cells around where I'm
working" — therefore lands on `[NOT_PARTITIONED]` before any real work starts,
and the caller has to detour (fetch `editor.open_level` docs, then find and open
a known-WP map) to recover.

## What it should do

Add a short **Preconditions** note to `docs/wiki-src/world_partition.md` (and,
because it is the most common entry point, surface it on the `load_cells` method
page): state that the active world must be world-partitioned, that the default
startup map is not, and give the two ways to get a WP world active
(`editor.open_level <WP map>` for an existing one, or
`level.structure.create_level {bCreateWorldPartition:true}` for a new one), plus
the `get_level_structure_info.worldPartitionEnabled` readback to check first.
This is a discovery fix only — the `[NOT_PARTITIONED]` error itself is correct
and the verbs all work once a WP map is active.

## Why this is distinct from the existing WP tickets

- `F-enable-world-partition-impossible` / `E-level-structure-wp-wiki-advertises-dead-end`
  are about the `level.structure` **enable/convert** path (code + that page's
  "typical flow"). This ticket is purely the `world_partition` namespace overlay
  page omitting the active-map precondition; different page, different verbs, and
  here the namespace itself works fine once a WP map is open.
- `B-configure-world-partition-silent-noop` / `E-perf-wp-configure-readback-thin`
  are the `performance.configure_world_partition` CVar-echo defect — different
  method, different namespace.
- `B-level-structure-info-datalayers-stub` is the empty `dataLayers` readback —
  different verb/output; it is in fact the readback this ticket would cite for the
  "check WP state first" hint, but the friction here is the missing precondition
  doc, not that readback's stub.

## Evidence (from the audited task — `world_partition` data-layer setup, 17 calls)

Outcome was **clean** (task succeeded), but with avoidable process friction at the
very start. Friction note, verbatim:

> "Minor: the initially-loaded active map was NOT world-partitioned, so the first
> load_cells returned [NOT_PARTITIONED]; I had to discover and open a WP map
> (Level_WorldPartitionStreaming) before the workflow could proceed. Everything
> else was smooth…"

Call-log shows the cost: `world_partition.load_cells {origin[0,0,0]
extent[50000^3]}` → `[NOT_PARTITIONED] World is not partitioned.` (call #4), then a
recovery detour — `editor.open_level` (args-omitted, to fetch the doc), then
`editor.open_level /Game/Maps/Level/Level_WorldPartitionStreaming` — before the
retry `load_cells` succeeded (call #7). Three calls (one failed verb + a doc fetch
+ the open) spent re-establishing a precondition the namespace wiki could have
stated up front. The auditor noted the rest of the workflow was smooth and the
per-method docs were clear once the WP map was active — the gap is solely the
namespace-level precondition.

## Fix (wiki overlay — `docs/wiki-src/world_partition.md`)

Add a **Preconditions / First steps** block to the namespace overlay (it
currently has only the one description line): "All `world_partition` verbs act on
the active editor world and require it to be a World Partition map; the default
startup map is not partitioned. Make a WP world active first —
`editor.open_level <existing WP map>` or
`level.structure.create_level {bCreateWorldPartition:true}` — and confirm with
`level.structure.get_level_structure_info` (`worldPartitionEnabled:true`).
Calling a verb on a non-WP world returns `[NOT_PARTITIONED] World is not
partitioned.`" Mirror a one-line version onto the `load_cells` method page as the
most common entry point.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of a `world_partition`
  data-layer setup task (17 calls; outcome clean — the judge filed nothing as
  there was no tool bug). PROCESS finding: the `world_partition` namespace wiki
  overlay (`docs/wiki-src/world_partition.md`, a one-line stub) never states that
  its verbs require an active world-partitioned map and that the default startup
  map is non-WP, so the natural opening call `world_partition.load_cells` (call #4
  in the log) returned `[NOT_PARTITIONED] World is not partitioned.`, forcing a
  three-call recovery detour (failed verb → `editor.open_level` doc fetch → open
  `Level_WorldPartitionStreaming`) before the workflow could start. Proposed
  remedy: add a Preconditions note to the overlay (and the `load_cells` page)
  pointing at `editor.open_level <WP map>` / `create_level {bCreateWorldPartition:true}`
  and the `get_level_structure_info.worldPartitionEnabled` readback. Dedup:
  ripgrep over OPEN + closed — distinct from `F-enable-world-partition-impossible`
  / `E-level-structure-wp-wiki-advertises-dead-end` (the `level.structure`
  enable/convert page, not this namespace), `B-configure-world-partition-silent-noop`
  / `E-perf-wp-configure-readback-thin` (`performance.configure_world_partition`),
  and `B-level-structure-info-datalayers-stub` (empty `dataLayers` readback). No
  existing ticket targets the `world_partition` overlay's missing active-map
  precondition.
- `#2-additional-create-datalayer-entry` `OPEN` reporter — Additional evidence
  (independent audit, data-layer "Decorations" setup task, 21 calls, seed
  `world_partition.set_datalayer`): same precondition gap, hit via a DIFFERENT
  verb. Here the natural opening WP call was `world_partition.create_datalayer
  {dataLayerName:"Decorations"}` against the default-loaded non-WP map
  (`ExampleProjectWelcome`), which returned `[NOT_PARTITIONED] World is not
  partitioned.` Recovery cost the same detour: Glob the project's `.umap` files to
  find a WP map, fetch `editor.open_level` docs, then `editor.open_level
  /Game/Maps/Level/Level_WorldPartitionStreaming`, after which create/set/load/
  cleanup all worked. Attempt-agent friction verbatim: "create_datalayer first
  failed with [NOT_PARTITIONED] because the editor's open map was the
  non-partitioned ExampleProjectWelcome … The error message named the cause but
  gave no hint to open a partitioned map." Confirms the gap is namespace-wide (not
  just `load_cells` from #1) — `create_datalayer` is the most common authoring
  entry point and needs the precondition note too, supporting the proposed
  namespace-overlay fix. Seed method itself replay-confirmed CLEAN: on the active
  WP map, `create_datalayer {OracleReplayLayer}` → created, `set_datalayer
  {BP_DemoDisplay13 → OracleReplayLayer}` → `added:true`, and `actor.describe`
  shows the actor's `DataLayerAssets` = `[/Game/DataLayers/Decorations.Decorations,
  /Game/DataLayers/OracleReplayLayer.OracleReplayLayer]` — assignment persists
  correctly. No tool bug; only the discoverability gap already tracked here.
