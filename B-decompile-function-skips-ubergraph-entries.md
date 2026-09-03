---
id: B-decompile-function-skips-ubergraph-entries
title: "blueprint.decompile_function only searches FunctionGraphs, can't reach EventGraph event overrides / custom events"
status: WONTFIX
severity: Medium
category: bug
tags: [bpir, decompile, eventgraph, ubergraph, event-override, custom-event, round-trip]
---

# blueprint.decompile_function only searches FunctionGraphs, can't reach EventGraph event overrides / custom events

`FBpirDecompiler::DecompileFunction` (`Private/Decompiler/BpirDecompiler.cpp:392-413`) iterates only `TargetBlueprint->FunctionGraphs` when looking up `FunctionName`. It never inspects `UbergraphPages`, so any entry that lives on the EventGraph — event overrides (`OnInitialized`, `ReceiveTick`, `Construct`), engine-binding events, and custom events declared in the EventGraph — return `[DECOMPILE_FAILED] DecompileFunction: function 'X' not found`.

The sibling method `DecompileGraph` (same file, lines 365-368) already does the right thing: it tries `FindByName(UbergraphPages)` first, then falls through to `FindByName(FunctionGraphs)`. `DecompileFunction` is asymmetric for no apparent reason.

## Repro

- Asset: `/App/App/UI/LobbyAndMenu/Elements/W_AnalyzerControls` — its EventGraph contains an `entry override OnInitialized() @(0, 15) {...}` block (confirmed by full `blueprint.decompile`).
- `mcp__editor-automation__call` with `path=blueprint.decompile_function`, `args={"assetPath":"/App/App/UI/LobbyAndMenu/Elements/W_AnalyzerControls","functionName":"OnInitialized"}` → `[DECOMPILE_FAILED] DecompileFunction: function 'OnInitialized' not found`.
- `blueprint.decompile` on the same asset emits the OnInitialized entry under `# ==== Graph: EventGraph (ubergraph) ====`.

## Impact

The only round-trip workflow for one event handler is full-BP `blueprint.decompile` + manual text-grep / sed extraction. Over-returns content, awkward when iterating on a single event handler, and forces every caller to know how to slice ubergraph entries out of the dump.

**Workaround:** Call `blueprint.decompile` (or `blueprint.decompile` with `graphName:"EventGraph"`) and grep the output for `entry (override|event|custom event) <name>(`.

**Fix:** In `DecompileFunction`, search `UbergraphPages` first (matching `DecompileGraph`'s order at line 365), then fall back to `FunctionGraphs`. When the match is in the ubergraph, narrow the decompilation to only the single entry node whose name matches (event override / custom event), not the whole ubergraph — `DecompileGraphInternal` already collects per-entry texts, so filter the returned `EntryTexts` to the requested name. Alternatively, add a sibling `blueprint.decompile_event` with the same `(assetPath, functionName)` shape that targets only ubergraph entries; the function/event split would mirror `UEdGraphSchema_K2`'s own graph-kind distinction.

## History
- `#1-ubergraph-not-searched` `OPEN` reporter — Verified handler at `BlueprintDecompilerHandler.cpp:85-115` calls `Decompiler.DecompileFunction(FunctionName)`. `BpirDecompiler.cpp:392-413` iterates only `TargetBlueprint->FunctionGraphs`. Repro on `/App/App/UI/LobbyAndMenu/Elements/W_AnalyzerControls` with `functionName:"OnInitialized"` returned `function 'OnInitialized' not found` even though full-BP `blueprint.decompile` emits the entry under `EventGraph (ubergraph)`. Sibling `DecompileGraph` at line 365 already searches `UbergraphPages` then `FunctionGraphs` — the asymmetry looks like an oversight rather than an intentional design split.
- `#2-wontfix-workaround-acceptable` `WONTFIX` developer — Existing workaround (call `blueprint.decompile` on the whole BP and grep, or read the asset-dump cache at `.editor-automation/asset-dumps/`) is acceptable. Narrowing `DecompileFunction` to a single ubergraph entry introduces a new name-match surface (raw FName `ReceiveTick` vs BPIR-emitted `Tick`, plus multi-token signatures for bound delegate events) without a clear win for callers. Closing without code change.
- `#3-error-message-does-not-redirect` `WONTFIX` reporter — Hit again 2026-09-02 on `/Game/FPS/UI/Test/BP_HUDTestPawn` (`functionName:"RunCycle"`, a custom event) and `/Game/FPS/UI/BP_HUDManager` (`functionName:"EventGraph"`), both `[DECOMPILE_FAILED] ... not found`. Not disputing the WONTFIX — the workaround is fine once you know it. The remaining cost is purely the **error message**, which asserts the entry does not exist when it does, and names no alternative: it prints `DecompileFunction: function 'X' not found` and a Docs link to `blueprint.decompile_function.md`, whose page lists only `assetPath` and `functionName` and never mentions that EventGraph entries live in a different verb. Two calls and a wiki read were spent before falling back to `blueprint.decompile`. Cheap ergonomic fix inside the existing WONTFIX: when the name misses `FunctionGraphs`, probe `UbergraphPages` for the same name and, on a hit, fail with something like `function 'X' is an EventGraph entry — use blueprint.decompile (optionally graphName:"EventGraph") and select the 'entry ... X(' block`. No new name-match surface is exposed, because the probe only chooses the wording of an error that is returned either way.
