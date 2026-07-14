---
id: B-orphan-sweep-mathexpression-fatal-load
title: "Deleting a collapsed-Tick MathExpression's orphans leaves a half-initialized composite that passes compile+save but FATALLY asserts on next cold load"
status: OPEN
severity: Critical
category: bug
tags: [orphan-sweep, delete_orphaned_nodes, remove_event, math-expression, composite, k2node, package-corruption, fatal-assert, load-time]
encounters: 1
lastSeen: 2026-07-14T17:00:00Z
---

# Orphan sweep can leave a broken K2Node_MathExpression shell → fatal assert on package load

## What happened (live incident)

During a widget refactor session, three widget BPs (`W_DronStatsLeft`, `W_DroneStatsRight`,
`W_DroneStatsRightSimple`) had their BP `Tick` deleted (`blueprint.remove_event`) and leftover
"pure orphans from the old Tick chain" swept with `blueprint.graph.delete_orphaned_nodes`. The agent
reported the leftovers as "collapsed-Tick MathExpression subgraph leftovers (float+float, float/float,
tunnel orphans in a MathExpression graph)". All three widgets **compiled `errors: []` and saved**,
and their refreshed bpir dumps showed no reachable math-expression logic.

The corruption detonated only later, on the next editor session's **first load** of any of the three
packages (or anything embedding them — `W_HUD_Common` embeds two, so the OSD became unloadable):

```
Assertion failed: OutputSourceNode [K2Node_Composite.cpp] [Line: 350]
```

Callstack: `PinWright!LoadBlueprintAsset → StaticLoadObject → linker CreateExport →
regenerate-on-load → FBlueprintEditorUtils::ReconstructAllNodes →
UK2Node_MathExpression::ReconstructNode → RebuildExpression → ClearExpression →
UK2Node_Composite::GetExitNode() → check(OutputSourceNode)`.

Binary scan confirmed: the three saved uassets contain a `K2Node_MathExpression` export with **no
expression text** and (in one) leftover `NewMacro`/`TempGraph` tunnel exports; the HEAD versions
contain none. So the sweep (or the Tick removal) deleted the composite's inner entry/exit tunnels
but left the outer MathExpression node in the package — a state that is invisible to
`blueprint.compile`, invisible to the dumps, and **fatal at load-time reconstruction**, because
`UK2Node_MathExpression::ReconstructNode` unconditionally walks its inner graph's exit node.

Recovery required reverting the three uassets to HEAD and re-applying the whole refactor — with the
editor crashed and the project unbootable until the revert.

## Why this is Critical

- The failure is **silent at write time** (compile green, save green, dump clean) and **fatal at
  read time** — the worst possible ordering. An agent can corrupt a package, verify it by every
  available means, report success, and brick the next session.
- Blast radius is transitive: any asset whose tree embeds the broken widget becomes unloadable too.

## Proposed fixes (any subset)

1. **`delete_orphaned_nodes` (and `remove_event`) must treat composite-family nodes as units**:
   when a sweep would delete a `K2Node_Tunnel` that is the entry/exit of a
   `K2Node_MathExpression`/`K2Node_Composite`/collapsed graph, delete the OWNING composite node
   (which tears down its inner graph properly via the editor API) instead of the tunnels.
2. **Post-mutation package validation**: after any graph mutation batch, walk the graph for
   composite nodes whose `GetEntryNode()`/`GetExitNode()` resolve null and either auto-remove them
   (via `FBlueprintEditorUtils::RemoveNode`) or fail the RPC with a clear error BEFORE save.
3. **Defensive `LoadBlueprintAsset`**: PinWright's own load path could pre-scan for null-exit
   composites and strip them before regenerate-on-load runs (fix-forward for already-corrupt
   packages) — lower priority than 1/2 but turns "unbootable project" into "logged repair".

## Acceptance

- Repro: create a widget BP with a Tick containing a MathExpression; `remove_event` Tick;
  `delete_orphaned_nodes`; save; **cold-load the package in a fresh editor** — must load cleanly
  (node either fully removed or fully intact).
- The validation (fix 2) fires on the repro if fix 1 is absent.

## History

- `#1-filed` **OPEN** (Reporter) — Filed from the live incident above: three widgets corrupted by a
  Tick-removal + orphan-sweep session, fatal `check(OutputSourceNode)` on next cold load, project
  unbootable until git revert. Crash folders `UECC-Windows-84CE7A7C4AC39BADB99BED8CBCE1E47D_0000`
  (load-time assert) and `UECC-Windows-0D3E0A7E45EE1326D21447B3FAF64F1E_0003` (the same session's
  earlier `WidgetVariableNameToGuidMap.Contains` ensure in WidgetBlueprintCompiler.cpp:794 — possibly
  related, worth checking while fixing).
