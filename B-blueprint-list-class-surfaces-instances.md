---
id: B-blueprint-list-class-surfaces-instances
title: "blueprint.list with a native class filter surfaces in-memory level actor INSTANCES instead of authorable Blueprint assets"
status: OPEN
severity: Medium
category: bug
tags: [blueprint-list, class-filter, world-partition, include-only-on-disk, asset-registry]
encounters: 1
lastSeen: 2026-07-02T11:19:56Z
---

# `blueprint.list` with a native class filter surfaces in-memory level actor INSTANCES instead of authorable Blueprint assets

`blueprint.list` is documented as "Enumerate blueprint **assets** with filters and
paging," `class` = "Blueprint class filter (default: Blueprint)." Passing the
natural value for "existing actor Blueprints" — `class="Actor"` — returned rows that
are **world-placed actor instances**, not authorable Blueprint assets, e.g.:

```
name "LocationVolume_UAID_025041000001F96C02_1283462387"
path "/Game/Maps/Level/Level_WorldPartitionStreaming.Level_WorldPartitionStreaming:PersistentLevel.LocationVolume_..."
name "LandscapeStreamingProxy_..."  class "/Script/Landscape.LandscapeStreamingProxy"
```

The `:PersistentLevel.` object paths are the tell: these are **in-memory sub-objects
of the open (World Partition) level**, the exact same phenomenon fixed in
`B-dump-folder-includes-level-subobjects` (DONE). The result overflowed the display
budget (46156 chars, spilled to a file) and the agent abandoned the RPC entirely,
falling back to a filesystem `Glob "Content/**/BP_*.uasset"` to find the real target
(`BP_Door`).

## Root cause (source-confirmed)

`PinWright_BlueprintHandlers_List.cpp:135-137` builds the `FARFilter` with
`bRecursivePaths`/`bRecursiveClasses=true` but **never sets
`bIncludeOnlyOnDiskAssets`**. When a level is loaded, `IAssetRegistry::GetAssets`
returns in-memory level actor instances as `FAssetData` alongside real on-disk
packages. The default `class="Blueprint"` (`:151-154`) dodges this — no UBlueprint
instances are placed in levels — but any native class filter (`:155-169`, here
`Actor`, matched recursively) pulls in every placed actor instance whose class
descends from it. The BP parent-tag walk (`:174-179`) correctly adds the *authorable*
BP subclasses, so the returned list is a **mix** of real BP assets and level
instances, with the instances dominating.

## What it should do

Set `Filter.bIncludeOnlyOnDiskAssets = true` before `GetAssets` (the one-line fix
`B-dump-folder-includes-level-subobjects` shipped for `asset.dump_folder`); optionally
also drop any result path containing `:` (the sub-object separator) as belt-and-
suspenders. Then a native-class `blueprint.list` returns only authorable Blueprint
`.uasset` rows, matching the "Enumerate blueprint assets" contract.

## Distinct from

- `B-dump-folder-includes-level-subobjects` (DONE) — **same root cause and same
  one-line fix**, but on `asset.dump_folder`; `blueprint.list` was never patched and
  still omits the flag. Filed separately because it is a different handler/method.
- `E-blueprint-list-no-projection-spills` (OPEN) — same method, but that ticket is the
  per-row **response-size** projection gap; this ticket is **wrong-kind rows** (level
  instances). The overflow here is a symptom the projection ticket would shrink but not
  correct — even trimmed, the rows would still be instances.
- `B-blueprint-list-class-ensure` (DONE) — same method, short-class ensure crash;
  orthogonal.
- `F-asset-search-native-subclass` (DONE) — added `parentClassPath` to `asset.search`
  (and blueprint.list's native branch) for finding BP subclasses; that fix is the
  parent-tag *walk* (adds the wanted assets). This ticket is the *complementary* leak:
  the raw FARFilter half still lets in-memory instances through because the on-disk
  flag is missing. (That ticket's `#3` tester already noticed `parentClassPath` returning
  a `:PersistentLevel.` `B_Race_C_1` instance — the same leak, unresolved.)

## Evidence

From the struggle audit of a clean `blueprint.undo_last_compile` task (namespace
`blueprint`, outcome clean, 12 RPCs). CallAnalyzer transcript
`agent-addbb09ce332e110a.jsonl`: `blueprint.list {class:"Actor", limit:60}` returned
`outputTooLong` (46156 chars, written to file); the Read showed the results dominated
by `:PersistentLevel.` level instances (LocationVolume, LandscapeStreamingProxy); SAY:
"The list is dominated by placed level-actor instances, not authorable BP assets. Let
me look at the project's Content on disk" -> abandoned the RPC for a Glob fallback.
Root cause verified in `PinWright_BlueprintHandlers_List.cpp:135-137` (no
`bIncludeOnlyOnDiskAssets`).

severity rationale: impact=soft-blocker returning wrong-kind data on a normal path (workaround: abandon the RPC, fall back to Glob or manually discard `:`-path rows) × reach=native-class filter is the natural first attempt for "list actor Blueprints" and the leak fires whenever a level is open (the normal editor state), though the default class="Blueprint" path is unaffected -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean `blueprint.undo_last_compile` task (12 RPCs, all `ok`, zero retries). Friction: the opening discovery call `blueprint.list {class:"Actor", limit:60}` returned `outputTooLong` (46156 chars, spilled to file); reading it showed the rows dominated by in-memory World-Partition level actor instances with `:PersistentLevel.` object paths (LocationVolume, LandscapeStreamingProxy), not authorable BP assets, forcing the agent to abandon the RPC and fall back to a filesystem `Glob "Content/**/BP_*.uasset"` to find `BP_Door`. Source-confirmed root cause: `PinWright_BlueprintHandlers_List.cpp:135-137` builds the `FARFilter` without `bIncludeOnlyOnDiskAssets=true`, so a loaded level's in-memory actor instances leak into any native-class (recursive) filter; default `class="Blueprint"` dodges it. Same root cause + one-line fix as `B-dump-folder-includes-level-subobjects` (DONE, `asset.dump_folder`), which `blueprint.list` never got. Dedup: ripgrep across OPEN/DONE — `E-blueprint-list-no-projection-spills` (response-size, not wrong-kind), `B-blueprint-list-class-ensure` (short-class crash), `F-asset-search-native-subclass` (parent-tag walk on a different method) are all distinct. Proposed: set `Filter.bIncludeOnlyOnDiskAssets = true` (+ optional `:`-path drop).
</content>
