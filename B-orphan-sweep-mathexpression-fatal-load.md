---
id: B-orphan-sweep-mathexpression-fatal-load
title: "Deleting a collapsed-Tick MathExpression's orphans leaves a half-initialized composite that passes compile+save but FATALLY asserts on next cold load"
status: IN-REVIEW
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
- `#2-additional-adversarial-review-a` **OPEN** (Reporter) — **Adversarial review A:** Current, not
  stale. `delete_orphaned_nodes` and `remove_event`/`CleanupNewBlueprintOrphans` still call
  `FBlueprintEditorUtils::RemoveNode` independently on orphan descriptors; the integrity gate has no
  `K2Node_Composite`/`K2Node_MathExpression` boundary check. Existing orphan and BPIR-expression tests
  do not exercise a saved package through fresh-editor load. The critical write-time false-success /
  fatal cold-load framing is sound. Fix 1 must discover and deduplicate the owning composite before
  deleting any tunnel, including nested bound graphs. Fix 2 should fail and roll back before save;
  auto-repair is risky. Fix 3 cannot pre-scan through `LoadBlueprintAsset`: `LoadObject` reconstructs
  during load, before that function can inspect the asset. Acceptance must save to disk, restart a fresh
  editor, load the package, and assert the composite is fully gone or intact.
- `#3-additional-boundary-sweep-review` `OPEN` reporter — Additional evidence: **Adversarial review B — KEEP; agrees with A that the corruption vector is current, but disagrees with limiting the fix scope to `delete_orphaned_nodes` and `remove_event`.** Actuality: CONFIRMED CURRENT. Framing: Critical crash/corruption is accurate; no duplicate found (the DONE `E-orphan-delta-cleanup` established delta behavior, not composite safety), but the historical widget repro is not freshly runtime-verified. Proposed fix: SYSTEMIC, centralize boundary-aware deletion that deduplicates an owning `UK2Node_Composite` before removing any tunnel, validates all composites, cancels the transaction before save on failure, and checks/reports save outcome; cover `delete_orphaned_nodes`, `CleanupNewBlueprintOrphans` (including `delete_node`/replace cleanup), and `remove_event`. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphOrphanHandler.cpp:125` removes each orphan independently; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphOrphanHandler.cpp:131` compiles/saves without checking the outcome; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintHandlerUtils.cpp:2694` repeats raw orphan removal; `C:\UE_5.8\Engine\Source\Editor\BlueprintGraph\Private\K2Node_Tunnel.cpp:59` nulls the composite back-pointer; `C:\UE_5.8\Engine\Source\Editor\BlueprintGraph\Private\K2Node_Composite.cpp:350` asserts on null `OutputSourceNode`; `C:\UE_5.8\Engine\Source\Editor\BlueprintGraph\Private\K2Node_MathExpression.cpp:2858` reconstructs during load. Runtime: NOT VERIFIED. Recommendation: KEEP; implement the shared guard and require a persisted MathExpression repro, fresh-editor cold-load, and explicit intact/removed assertion.
- `#4-boundary-tunnel-never-swept` `IN-REVIEW` developer — Root cause is narrower than reviews A/B assumed and is entirely inside the *scan*, not the delete loop. `ScanGraphForBlueprintOrphans` skips entry tunnels (`IsBlueprintEntryNode` → `IsMacroEntryTunnel`) but had no mirror for exit tunnels, and `CollectAllBlueprintGraphsRecursive` descends into composite bound graphs. In a `UK2Node_MathExpression` bound graph the entry tunnel has no exec pins, so `BuildExecReachabilitySet` stops there and `BuildFullReachabilitySet`'s backward data walk (input pins only) reaches nothing — the exit tunnel *and* every generated math node came back as orphans. `UK2Node_Tunnel::DestroyNode` (`K2Node_Tunnel.cpp:52-59`) then nulls `OutputSourceNode` on the surviving composite. The composite survives because `BuildExecReachabilitySet` seeds any composite whose bound graph holds an entry-point node, and a bound graph *always* holds its entry tunnel — so composites are never reported as orphans at all, which is exactly the observed "outer MathExpression export left behind, inner tunnels gone". Fatal path is engine-side and is an **assert, not a recoverable error**: `UK2Node_MathExpression::ReconstructNode` (`K2Node_MathExpression.cpp:2854`, guarded only by `RF_NeedLoad`, so regenerate-on-load enters it) → `RebuildExpression` → `ClearExpression` (`:2741`) → `GetExitNode()` → `check(OutputSourceNode)` (`K2Node_Composite.cpp:348-351`). `ClearExpression` calls `GetExitNode()` *before* the `Expression.IsEmpty()` branch, which is why the empty-expression export in the incident still detonated. Fixed in `BlueprintHandlerUtils.cpp`: (a) `ScanGraphForBlueprintOrphans` now skips `IsMacroExitTunnel` nodes — a boundary, never sweepable, in both `includeDataOnly` modes; (b) `BuildFullReachabilitySet` seeds the graph's exit tunnel as a data root so the generated expression nodes are correctly reachable, giving the ticket's "fully intact" rather than a hollowed-out node (the `Reachable.Num() == 0` fast-path was dropped so the seeding still runs on an entry-less bound graph). One scan-level fix covers every deleter, since `delete_orphaned_nodes` and `CleanupNewBlueprintOrphans` (→ `remove_event`, `delete_node`, replace-cleanup, widget hierarchy) all source descriptors from `FindBlueprintOrphanNodes`; review B's plumbing of a failure path through all five was therefore not needed. Fix 2 added narrowly as defense-in-depth: `DeleteOrphans` runs `FindBrokenCompositeBoundary` before compile+save and, on a null `InputSinkNode`/`OutputSourceNode`, calls `Transaction.Cancel()`, skips the save and returns new `COMPOSITE_BOUNDARY_BROKEN` (registered in `Handlers/ErrorCodes.h`) — the guarantee it makes is "nothing was written", not "rollback always succeeded. Fix 3 confirmed impossible as review A said: `LoadObject` reconstructs during load, before `LoadBlueprintAsset` can inspect anything. Regression test `PinWright.blueprint.graph.delete_orphaned_nodes.MathExpressionBoundaryStaysIntact` builds a real `UK2Node_MathExpression` (`Expression = "a + b"`), sweeps, and asserts the ticket's disjunction — node gone, or tunnels non-null and bound-graph node count unchanged — reading `InputSinkNode`/`OutputSourceNode` directly rather than through the check()-guarded accessors, and never reloading a package; a cold-load repro was deliberately not written, since the failure it would prove is a hard editor kill, not a catchable test failure. **Not compiled and not run** (shared checkout, build forbidden this pass) — needs a suite run before DONE. Follow-up worth its own ticket, not fixed here: composites are permanently unsweepable for the seeding reason above, so a genuinely dead collapsed graph is never removed; fixing it means changing `IsBlueprintEntryNode`/`CollectEntryNodesRecursive`, which `get_execution_flow` and the decompiler also depend on.
