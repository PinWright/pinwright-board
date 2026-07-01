---
id: B-anim-graph-search-cdo-ensure
title: "animation.list_graph_nodes / animation.search_graph_nodes ensure on AnimGraphNode CDOs (GetNodeTitle dereferences a graph the CDO lacks)"
status: IN-REVIEW
severity: Low
category: bug
tags: [anim-graph, search, discovery, ensure, cdo, getnodetitle, crash-report]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# animation.list_graph_nodes / animation.search_graph_nodes ensure on AnimGraphNode CDOs — GetNodeTitle dereferences a graph the class-default object doesn't have

The anim-graph-node discovery catalog (the `animation.list_graph_nodes` and
`animation.search_graph_nodes` verbs, both routed through the same `BuildEntry`)
builds one entry per `UAnimGraphNode_Base` descendant by reading fields off each
class's **class-default object (CDO)**. For the `description` field it calls
`GetNodeTitle(ENodeTitleType::ListView)` on the CDO. For the `ListView` title
type, `UAnimGraphNode_SaveCachedPose` (the one stock node confirmed) does **not**
short-circuit: its title path routes through `FNodeTextCache` →
`UEdGraphNode::GetGraph()`, which **ensures** the node has a `UEdGraph` as its
`Outer`. A CDO is outered to its class/package, not to a graph, so the ensure
fires:

```
Ensure condition failed:
EdGraphNode::GetGraph : '/Script/AnimGraph.Default__AnimGraphNode_SaveCachedPose'
does not have a UEdGraph as an Outer.
```

It is a **non-fatal ensure** (the editor survives and the suite still reports
green). Its single real consequence is **crash-report spam**: every catalog
build trips the ensure once per affected node-type *call*, writing a UE crash
dump each time. Observed as a cluster of dumps under `Saved/Crashes/` on every
editor test cycle that exercises the handler (the `TestAnimGraphHandlers`
automation hits it each run). On a live agent session, a single
`animation.list_graph_nodes` / `animation.search_graph_nodes` call would
likewise emit an ensure / crash report — alarming and noisy on a read-only
discovery verb.

The `description` field itself is **not** degraded: `FNodeTextCache::SetCachedText`
populates the cached title *before* the `GetGraph()` ensure (EdGraphNodeUtils.h),
so `GetNodeTitle` still returns the correct text (`"Save cached pose ''"` for the
empty-`CacheName` CDO). The harm is purely the crash-dump noise — hence severity
**Low** (a non-fatal ensure that blocks nothing and corrupts no data; the method
returns correct results without any workaround).

## Root cause (verified in source)

`Handlers/Animation/AnimGraphSearchHandler.cpp`, `BuildEntry(UClass* C)` (handler
namespace helper at `:33`):

```cpp
UAnimGraphNode_Base* CDO = Cast<UAnimGraphNode_Base>(C->GetDefaultObject());  // :37
if (CDO)
{
    E.Category    = CDO->GetMenuCategory().ToString();                          // :40 (graph-independent, fine)
    E.Description  = CDO->GetNodeTitle(ENodeTitleType::ListView).ToString();    // :41 <-- ensure fires for some node CDOs
    ...
```

`GetMenuCategory()` (:40) is graph-independent and safe. `GetNodeTitle(ListView)`
(:41) is **not** — for node types whose `ListView` title consults the owning
graph (e.g. cached-pose save/use nodes), it routes through `GetGraph()` and
ensures on a CDO that has no graph outer. The handler iterates every
`UAnimGraphNode_Base` descendant's CDO, so it trips on each such type.

This is a latent defect in a **shipped** feature — `F-search-api-anim-graph-nodes`
is `DONE`, and `AnimGraphSearchHandler.cpp` was last modified on 2026-06-23
(plugin-rename commit), so this is **not** a regression from any recent fix; it
has been present since the search API landed.

## What it should do

Read a graph-independent title for the CDO. Options, cheapest first:

- Use `GetNodeTitle(ENodeTitleType::MenuTitle)` instead of `ListView` — the
  menu title generally does not dereference the owning graph. (Verify per node
  type.)
- Or guard the call: skip/short-circuit the title for a CDO whose `GetGraph()`
  would be invalid, and fall back to the class display name
  (`C->GetDisplayNameText()` / a cleaned `GetName()`).
- Or wrap the `GetNodeTitle` call so the known graph-dependent node classes use
  the class display name and never reach `GetGraph()` on a CDO.

Any of these removes the ensure (and the crash-dump spam) and yields a correct,
stable `description` for every node type. Note: a regression test cannot rely on
"no ensure" auto-failing the suite (the ensure is non-fatal and the suite stays
green today), and "non-empty `description`" does not distinguish fixed from
broken (the field is non-empty in both states). The reliable signal is the
*value*: the reverted `ListView` path yields `SaveCachedPose` description
`"Save cached pose ''"`, whereas a graph-safe fix yields a different label — so a
test should assert the `SaveCachedPose` catalog `description` equals the
graph-safe title (computed live, e.g. via `MenuTitle`).

## Evidence

- Crash dumps: `Saved/Crashes/UECC-Windows-*` clusters (4 per affected cycle),
  `CrashContext.runtime-xml` → `"Ensure condition failed"` with the
  `Default__AnimGraphNode_SaveCachedPose … does not have a UEdGraph as an Outer`
  message. Most recent cluster observed 2026-06-25 ~08:03.
- Source: `Source/PinWright/Private/Handlers/Animation/AnimGraphSearchHandler.cpp:33-56`
  (`BuildEntry`), crash at `:41`.
- Exercised every cycle by `Source/PinWright/Private/Tests/Assets/TestAnimGraphHandlers.cpp`.

**Workaround:** none needed to *use* the method (it still returns correct
results — including a correct `description`); the ensure is non-fatal. But there
is no way to suppress the per-call crash-dump generation without the fix.

**Fix:** read a graph-independent title for the CDO (e.g. `MenuTitle`, or a
guarded fall back to the class display name) so `GetNodeTitle` never reaches
`GetGraph()` on a graph-less CDO.

## History
- `#1-crash-forensics` `OPEN` reporter — Surfaced via crash-dump forensics during an mcp-fix-workflow supervision pass (not a live MCP repro): ~88 UE crash dumps over 12h, all the same non-fatal ensure `EdGraphNode::GetGraph : '…Default__AnimGraphNode_SaveCachedPose' does not have a UEdGraph as an Outer`, clustered one-per-test-cycle. Root-caused in source: `AnimGraphSearchHandler.cpp` `BuildEntry` calls `CDO->GetNodeTitle(ENodeTitleType::ListView)` (:41) on each `UAnimGraphNode_Base` descendant's class-default object; for node types whose ListView title consults the owning graph (confirmed `UAnimGraphNode_SaveCachedPose`), that path hits `GetGraph()` which ensures a `UEdGraph` outer the CDO doesn't have. Non-fatal (suite stays green) but spams crash dumps every call/cycle and degrades the `description` field for the affected entries. Pre-existing, not a recent regression (file last touched 2026-06-23; feature `F-search-api-anim-graph-nodes` is DONE). Fix: use a graph-independent title for the CDO (`MenuTitle`, or a guarded class-display-name fallback) so `GetNodeTitle` never reaches `GetGraph()` on a graph-less CDO; add a regression test asserting no ensure over the full catalog and a non-empty `SaveCachedPose` description.
- `#2-menutitle-fix-and-reword` `IN-REVIEW` developer — Reworded first (the report's `search.anim_graph_nodes` verb does not exist — the registered methods are `animation.list_graph_nodes` / `animation.search_graph_nodes`, AnimGraphSearchHandler.cpp:85,122; dropped the false consequence #2 — `FNodeTextCache::SetCachedText` writes the cached title *before* the `GetGraph()` ensure (EdGraphNodeUtils.h:54-56), so `GetNodeTitle` returns the correct, non-empty `"Save cached pose ''"` and the `description` is **not** degraded; re-rated severity Medium→Low per the README rubric — non-fatal ensure, no crash, no corruption, no silent-wrong-data, nothing blocked; corrected the test-path citation to `Tests/Assets/`). Fix: `AnimGraphSearchHandler.cpp` `BuildEntry` now reads `CDO->GetNodeTitle(ENodeTitleType::MenuTitle)` instead of `ListView`, falling back to `C->GetDisplayNameText()` if the menu title is empty. Verified in engine source that the `MenuTitle` branch short-circuits to a graph-independent label **before** reaching `FNodeTextCache`→`GetGraph()` for every one of the 18 stock `UAnimGraphNode_*` types that use cached titles on their default (CDO) field values (incl. `SaveCachedPose`, whose empty-`CacheName` MenuTitle returns `"New Save cached pose..."`), so the ensure can no longer fire on the stock catalog. Regression test `FAnimSearchGraphNodesCdoTitleNoEnsureTest` (`Tests/Assets/TestAnimGraphHandlers.cpp`) dispatches the real `animation.list_graph_nodes` RPC and asserts the `SaveCachedPose` entry's `description` equals the live graph-safe `MenuTitle` value (which the test computes safely, never calling `ListView`, so the test itself never trips the ensure) — reverting `BuildEntry` to `ListView` makes the description `"Save cached pose ''"` ≠ the expected MenuTitle label, failing the test. Files: `Source/PinWright/Private/Handlers/Animation/AnimGraphSearchHandler.cpp`, `Source/PinWright/Private/Tests/Assets/TestAnimGraphHandlers.cpp`.
