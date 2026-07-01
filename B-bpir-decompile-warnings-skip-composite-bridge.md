---
id: B-bpir-decompile-warnings-skip-composite-bridge
title: "BPIR decompiler `warnings` array reports composite + downstream chain as orphans even after entry-point fix"
status: DONE
severity: High
category: bug
tags: [bpir, decompile, warnings, reachability, k2node-composite, k2node-tunnel, orphan-detection, false-positive, post-walk-pass]
---

# BPIR decompiler `warnings` array reports composite + downstream chain as orphans even after entry-point fix

`B-bpir-entry-points-skip-composite-subgraphs` (IN-REVIEW) fixed three independent entry-point collectors so `find_orphaned_nodes`, `get_execution_flow`, and `list_graphs` all surface composite-tunnelled events and stop reporting their bodies as orphans. The BPIR decompiler also picks up the inner events as proper top-level entries (`entry event Tick`, `entry event BeginPlay`, `entry override ReceiveAsyncPhysicsTick`) and emits their bodies correctly.

But the BPIR decompiler has its **own** orphan-detection post-walk pass (`Decompiler/BpirDecompiler.cpp:457-482`) that the sprint did not touch. That pass runs after every entry's `WalkExecChain` completes, then iterates `Graph->Nodes` and warns about any node not in `AllVisitedNodes`. Two structural reasons it still produces wrong warnings:

1. **`WalkExecChain` does not bridge composite/tunnel boundaries.** It walks `Pin->LinkedTo` and terminates at `K2Node_Tunnel` exit pins inside a composite's `BoundGraph`. The outer composite node is never reached — the walker started inside the body, went out via the exit tunnel, and stopped. Composite node itself stays unvisited.
2. **Anything downstream of the composite's outer output exec pins is unreachable from any in-graph entry.** No bridge from inner-tunnel-pin → outer-composite-output-pin in the walker, so the parent-graph chain wired to those outputs (e.g. `Sequence → Delay → Broadcast Message → reroute knots`) is never traversed and lands in the warnings.

This is **the same shape** as the gap caught in `BlueprintGraphHandler.cpp::BuildExecutionChain` during the parent ticket's spec-review iteration — except `BpirDecompiler` carries its own walker, not the one in `BlueprintGraphHandler`, so the iteration-2 bridge fix did not flow through here.

## Repro (live, post-recompile, this session)

Asset: `/App/HELIOS/Framework/HELIOS_BP`. Three sister tools agree the composite body is reachable:

- `blueprint.graph.find_orphaned_nodes` — `orphanedCount: 1` (the one orphan is in an unrelated function graph; not the composite chain).
- `blueprint.graph.get_execution_flow entryPointsOnly:true includeAllEntryPoints:true` — returns 19 entry points including `Event Tick`, `Event BeginPlay`, `Event Async Physics Tick` (all three live inside `Activation Node`).
- `blueprint.graph.list_graphs` — surfaces `Activation Node`, `Camera HUB Node`, and the `MathExpression` subgraph each with `parentGraphName`.

But `blueprint.decompile_bpir graphName:"EventGraph"` returns the events as proper entries **and** still emits this `warnings` array:

```
"Orphaned node not reachable from any entry point: Activation Node"
"Orphaned node not reachable from any entry point: Camera HUB Node"
"Orphaned node not reachable from any entry point: Broadcast Message"
"Orphaned node not reachable from any entry point: Sequence"
"Orphaned node not reachable from any entry point: Delay"
"Orphaned node not reachable from any entry point: Reroute Node"  (×6)
```

Two tools that should agree (`find_orphaned_nodes` and the decompiler's warning pass) disagree by ~12 entries — the same disagreement pattern the parent ticket called out as the agent-confidence-failure mode (no signal to second-guess; cross-tool fallback returns the same wrong answer).

There is also a **deeper text-emission gap** lurking behind the warnings: the parent-graph chain wired to the composite output pins (`Activation Node.OnTick → Sequence → Delay → ...`) is currently **not emitted** in the BPIR text at all. The composite emits its own entry block (`entry event Tick { ... return [OnTick] }`), but the consumer of `OnTick` in EventGraph is invisible to the decompile output. The warnings are the symptom; the missing emission is the underlying cause.

## Impact

Severity rationale (High):

- The decompiler's `warnings` field is a primary signal MCP agents use to assess BPIR-text completeness. A non-empty warnings array implies "trust the BPIR less".
- Three sister tools now disagree with the decompiler. An agent's natural cross-check no longer protects them — `find_orphaned_nodes` says clean, decompiler says ~12 orphans, and the agent has no way to know which to believe without manually re-running both.
- The deeper emission gap (parent-graph chain not in BPIR text for composite-organised blueprints) means agents reading BPIR output to reason about `HELIOS_BP`-shaped blueprints get an incomplete picture of post-composite control flow.
- Not Critical because: the sister tools are correct, so a careful agent has a cross-check; and the BPIR text itself emits the inner entries correctly (the parent ticket's primary win) — the gap is in the warning sidecar plus the unwalked outer chain.

## Workaround

Use `blueprint.graph.find_orphaned_nodes` for orphan detection on composite-organised blueprints and ignore the decompiler's `warnings` array entries for nodes named after composites or downstream-of-composite reroute/Sequence/Delay knots. The handler is now the source of truth post-`B-bpir-entry-points-skip-composite-subgraphs`.

## Fix

Two layers, smallest first:

**(a) Patch the orphan-detection post-walk pass.** Add a reachability bridge after the entry walks complete, mirroring the composite-aware logic now in `BlueprintGraphOrphanHandler.cpp::BuildExecReachabilitySet`:
1. For each `K2Node_Composite` in `Graph->Nodes`: if any node inside `Composite->BoundGraph->Nodes` is in `AllVisitedNodes`, mark the composite itself visited.
2. From each newly-visited composite, BFS the parent graph via `Pin->LinkedTo` on its output exec pins, transitively marking downstream nodes visited (transparent over `K2Node_Knot` reroutes, same convention `WalkExecChain` already follows at lines 553-563). Stop at any node already visited.

This silences the false-positive warnings without touching `WalkExecChain` or BPIR text emission. ~30 lines, fully local to `BpirDecompiler.cpp`.

**(b) Bridge `WalkExecChain` itself.** Same change `BuildExecutionChain` received in iteration 2 of the parent ticket: when the walker reaches a `K2Node_Tunnel` exit pin, hop to the matching outer `K2Node_Composite` output pin (by name) and continue walking in the parent graph. This is the deeper correctness fix — it makes the BPIR text emit the parent-graph chain wired to composite outputs, not just suppress its warning. Bigger blast radius (changes BPIR text shape), should be planned not patched.

Recommend (a) immediately to align decompiler warnings with the other three tools; (b) tracked separately for a later sprint.

Regression test: in-memory blueprint with `K2Node_Composite` whose `BoundGraph` contains an `Event Tick` chain ending at an exit tunnel, plus a `K2Node_CallFunction` PrintString in the parent EventGraph wired to the composite's outer output pin. Expectation after fix (a): `blueprint.decompile` `warnings` array contains no entries naming the composite or its outer-graph downstream chain. Expectation after fix (b): the BPIR text body for the composite-rooted entry includes the outer PrintString call.

## History
- `#1-live-repro` `OPEN` reporter — Live verification of `B-bpir-entry-points-skip-composite-subgraphs` (IN-REVIEW) on `/App/HELIOS/Framework/HELIOS_BP` after plugin recompile. Three sister tools (`find_orphaned_nodes` → 1 orphan, `get_execution_flow entryPointsOnly:true includeAllEntryPoints:true` → 19 entries with all three composite-tunnelled events present, `list_graphs` → composite subgraphs surfaced with `parentGraphName`) confirmed the parent fix works. `blueprint.decompile_bpir graphName:"EventGraph"` correctly emits `entry event Tick`, `entry event BeginPlay`, `entry override ReceiveAsyncPhysicsTick`, and `entry macro Camera HUB Node()` — but the `warnings` array still reports `Activation Node`, `Camera HUB Node`, `Broadcast Message`, `Sequence`, `Delay`, and 6× `Reroute Node` as "Orphaned node not reachable from any entry point". Root cause traced to `Decompiler/BpirDecompiler.cpp:457-482` (post-walk orphan-detection pass) — `WalkExecChain` (line 524) walks `Pin->LinkedTo` but does not bridge across composite/tunnel boundaries, so the outer composite node and any chain downstream of its output pins are never marked visited. Same shape as the iteration-2 gap caught in `BuildExecutionChain` during the parent ticket; the fix landed there but `BpirDecompiler` has its own walker.
- `#2-minimal-bridge-applied` `IN-REVIEW` developer — Applied fix shape (a) from the ticket: added a composite/tunnel bridge pass in `Decompiler/BpirDecompiler.cpp` between the entry-walk accumulation (line 454) and the orphan-detection loop (now line 506+). Pass scans `Graph->Nodes` for any `K2Node_Composite` whose `BoundGraph` contains a node already in `AllVisitedNodes`, marks the composite visited, then BFS-walks the parent graph from each newly-visited composite's output exec pins via `Pin->LinkedTo`, marking downstream nodes visited transitively. Added `#include "K2Node_Composite.h"`. Walker (`WalkExecChain`) and BPIR text emission unchanged — fix is scoped to the warning sidecar only. Fix shape (b) — bridging `WalkExecChain` itself so the outer-graph chain wired to composite outputs is emitted in the BPIR text body — remains tracked in the **Fix** section above as a separate, larger change.
- `#3-verified-warnings-clean` `DONE` tester — Verified on the original repro asset `/App/HELIOS/Framework/HELIOS_BP`: `blueprint.decompile_bpir graphName:"EventGraph"` returned `success:true` with `warnings:["Unresolvable value: pin 'AttenuationSettings'…", "Unresolvable value: pin 'ConcurrencySettings'…"]` only — **zero** "Orphaned node not reachable from any entry point" warnings. Pre-fix the same asset produced 12 such warnings (Activation Node, Camera HUB Node, Broadcast Message, Sequence, Delay, 6×Reroute Node, Construction Script). The 5 emitted entries (`entry event Tick`, `entry event BeginPlay`, `entry override ReceiveAsyncPhysicsTick`, etc.) all surface from inside the composites correctly. Fix (a) (warning sidecar) verified; fix (b) (text emission of outer-graph chain) explicitly deferred per the developer note.
