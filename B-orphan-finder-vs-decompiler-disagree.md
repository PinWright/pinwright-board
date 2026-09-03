---
id: B-orphan-finder-vs-decompiler-disagree
title: "blueprint.graph.find_orphaned_nodes returns 0 while BPIR decompiler still reports orphan warnings on the same asset"
status: OPEN
severity: High
category: bug
tags: [orphan-detection, bpir-decompiler, find-orphaned-nodes, false-negative, data-only-orphans, includeDataOnly, false-clean, revived, weapons]
encounters: 2
lastSeen: 2026-09-03T00:00:00Z
---

# blueprint.graph.find_orphaned_nodes returns 0 while BPIR decompiler still reports orphan warnings on the same asset

After running `blueprint.graph.find_orphaned_nodes` (with `dryRun: false, includeDataOnly: true`) across all graphs of `/App/HELIOS/Framework/HELIOS_BP` and getting `orphanedCount: 0, deletedCount: 0`, an immediate `asset.dump diff:true` on the same asset still emits orphan warnings in the BPIR `# ==== Warnings ====` section:

```
Orphaned node not reachable from any entry point: Reroute Node      (×6)
Orphaned node not reachable from any entry point: Construction Script
```

The two tools have different definitions of "orphan":

- `blueprint.graph.find_orphaned_nodes` (graphsScanned: 16) reports 0.
- BPIR decompiler reports 7 (6 reroute knots + 1 "Construction Script").

This is not a regression introduced by the user's edits — it reduces from 14 in the baseline to 7 after a deletion sweep. But the residual 7 are invisible to `find_orphaned_nodes`, so a workflow that relies on the orphan finder as a "before save" gate (per the corruption-avoidance memory `feedback_mcp_bp_corruption_avoidance.md`) leaves BPIR-visible junk in the asset.

**Reproduce:**

1. On any non-trivial Blueprint that has dangling reroute knots or composite-graph residue, call `blueprint.graph.find_orphaned_nodes` with `dryRun: false, includeDataOnly: true` (no `graphName` to scan all).
2. Result: `orphanedCount: 0`.
3. Call `asset.dump <bp> diff:true` (or any decompile path that emits the `# ==== Warnings ====` block).
4. Result: warnings list contains "Orphaned node not reachable from any entry point" lines that point at nodes the finder did not see.

Confirmed on `/App/HELIOS/Framework/HELIOS_BP.HELIOS_BP` after a multi-step deletion of 9 functions and ~14 call/variable nodes. `find_orphaned_nodes` swept correctly during the cleanup (deleted 2 nodes in `Activation Node`), but the BPIR decompiler still finds 6 reroute knots and 1 "Construction Script" entry that the finder ignores.

## Root cause

Two reachability walks live in the plugin:
- `BlueprintGraphOrphanHandler::BuildExecReachabilitySet` (`Handlers/Blueprint/BlueprintGraphOrphanHandler.cpp:65`) — seeds from every `IsBlueprintEntryNode` plus composites with inner entries, BFS over output exec edges, marks knots reachable when linked. Correct.
- `FBpirDecompiler` orphan post-pass (`Decompiler/BpirDecompiler.cpp:506-530`) — derives "visited" from `WalkExecChain` (`Decompiler/GraphWalker.cpp`), which skips knots without recording them, then a pre-walk seed filter drops empty `UserConstructionScript::FunctionEntry`. Both omissions surface as false-positive "Orphaned node" warnings.

## Fix

Extract `BuildExecReachabilitySet` into `BlueprintHandlerUtils` (alongside `CollectEntryNodesRecursive`) returning `TSet<FGuid>`. Rewire the orphan handler to call the extracted helper. Replace the BPIR decompiler's orphan post-pass with a call to the same helper plus filters (`IsBlueprintEntryNode`, `UK2Node_Knot`, `UEdGraphNode_Comment`, `bHasExecPin`) so the warning list is computed from the canonical reachability set, not from the walker's emit-time bookkeeping. Also add the per-orphan `graphName` field to the single-graph response shape (`BlueprintGraphOrphanHandler.cpp:472`) so callers can target a fix without a `list_graphs` round-trip — the all-graphs path already includes it.

## Impact

- Workflows that rely on `find_orphaned_nodes` as a pre-save integrity gate (per `feedback_mcp_bp_corruption_avoidance.md` rule 3) get a false "all clean" signal. Junk knots persist in the saved `.uasset`.
- BPIR decompile output is noisier than necessary, mixing real residual orphans with previously-observable warnings.
- Differential diff workflows comparing baseline → post-edit struggle to distinguish "bug we cleaned up" from "bug that still lingers".

## Workaround

None reliable inside MCP today. The asset can be opened in the editor and the dangling knots manually deleted, then re-saved.

## History
- `#1-finder-misses-bpir-orphans` `OPEN` reporter — Reproduced on `/App/HELIOS/Framework/HELIOS_BP` after a multi-deletion sweep. `blueprint.graph.find_orphaned_nodes` returned `orphanedCount: 0` across 16 scanned graphs; immediately following `asset.dump diff:true` BPIR output had 6 "Orphaned node not reachable from any entry point: Reroute Node" warnings plus 1 "Construction Script" warning. Baseline pre-edit dump had 14 such warnings, post-edit has 7 — the finder swept some but not these. Inconsistency makes `find_orphaned_nodes` an unreliable pre-save integrity gate (cf. `feedback_mcp_bp_corruption_avoidance.md` rule 3).
- `#2-shared-reachability-walk` `IN-REVIEW` developer — Reshape: original ticket fingered the orphan finder; correct culprit is the BPIR decompiler's orphan post-pass which (a) does not register knots in `State.VisitedNodes` and (b) drops empty `UserConstructionScript::FunctionEntry` from seeds. Extracted `BuildExecReachabilitySet` into `BlueprintHandlerUtils`, rewired both the orphan handler and the decompiler post-pass to consume it; added `graphName` to single-graph orphan response. Tests: `LiveExecKnotNotFlagged` (TestBlueprintGraphOrphan.cpp) and `SkipsKnotAndEmptyConstructionScript` (TestBpirDecompilerOrphanWarnings.cpp).
- `#3-verified-tools-agree` `DONE` tester — Verified on `/App/HELIOS/Framework/HELIOS_BP` (the original repro asset): `blueprint.graph.find_orphaned_nodes` returned `orphanedCount: 0` across 16 graphs scanned. Same-asset `blueprint.decompile_bpir graphName:"EventGraph"` returned `warnings:["Unresolvable value: pin 'AttenuationSettings'…","Unresolvable value: pin 'ConcurrencySettings'…"]` — zero "Orphaned node not reachable from any entry point" warnings. Both tools now agree on 0 orphans (vs the original 0-vs-7 disagreement). Tools share the canonical `BuildExecReachabilitySet` reachability set.
- `#4-revived-data-only-orphans-missed` `OPEN` WEAPONS-critic — **Revived from DONE: the two verbs disagree again on the same asset, with the direction reversed, and severity raised Medium → High.** Measured during a WEAPONS critic review round 2 on `/Game/FPS/Weapons/Test/BP_WeaponTestPawn`. `call("blueprint.graph.find_orphaned_nodes", {"assetPath":"/Game/FPS/Weapons/Test/BP_WeaponTestPawn", "includeDataOnly": true})` → `{"totalNodes":60,"orphanedNodes":[],"orphanedCount":0}`. `call("asset.dump", {"assetPath":"/Game/FPS/Weapons/Test/BP_WeaponTestPawn"})` on the **same** asset writes into `bpir.txt`: `# BPIR_WARN: Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Is Valid' nodeId=9CC0E756-456B-B9C4-CD0F-2B8D75B942E2 @(0,3328)` plus a second at `59227870-4C46-6E62-FB56-08A7ECD4D7B7 @(240,5000)`. `blueprint.graph.get_node_details` confirms **both nodes exist with every pin unlinked**, so they are genuinely orphaned — this is not the decompiler over-reporting. **Direction is the opposite of what `#2` fixed:** `#1`/`#2` had the decompiler emitting false positives (reroute knots, empty construction script) that the finder correctly ignored, and the fix made the decompiler consume the finder's canonical `BuildExecReachabilitySet`. Here the **decompiler is right and the dedicated orphan verb is wrong** — called with `includeDataOnly: true`, the flag whose entire purpose this is, on two pure/data-only `K2Node_CallFunction` nodes. Both nodes are unreachable and fully unlinked, so any correct reachability walk should flag them. Severity raised to **High** on the silent-false-success band: the verb an agent calls specifically to certify a graph clean returns a clean answer over real orphans, on the normal path, and there is nothing in its response to prompt a second look — `#1`'s original impact was the noisier variant (extra warnings a caller could investigate), this is the variant where the caller stops looking. Concrete consequence in this project: the two orphans are the amputated stumps of the missing `StopFire` / `StopADS` handlers, so the false clean hid a real gameplay gap that a correct orphan report would have pointed straight at. Note for whoever picks this up: if the cause turns out to be a distinct code path — the finder's data-only/pure-node classification rather than the shared `BuildExecReachabilitySet` that `#2` introduced — splitting this into its own ticket and returning this one to DONE is a reasonable call; it is appended here because the defect statement in the title is verbatim what was re-observed, and filing a second ticket with the same title would be the duplicate the board asks reporters to avoid. No plugin source was read for this entry; the reversal above is inferred from the two responses and from `#2`'s own description of what it changed, and no `file:line` is claimed. Related, filed the same round: `B-bpir-disabled-nodes-emitted-as-live` — a second case of BPIR and the graph-inspection verbs disagreeing about `/Game/FPS/Weapons/` graphs, there with BPIR the one reading clean.
