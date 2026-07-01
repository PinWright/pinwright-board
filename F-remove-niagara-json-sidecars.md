---
id: F-remove-niagara-json-sidecars
title: "Remove niagara_graphs / niagara_model / niagara_stack / niagara_parameters / niagara_emitters / niagara_system JSON sidecars"
status: DONE
severity: Medium
category: feature
tags: [niagara, nir, dump-format, size-reduction]
---

# Remove the Niagara JSON sidecars (keep only niagara_compile.json)

The Niagara JSON sidecars dominate the asset-dump cache and duplicate semantics that `nir.txt` already represents in 1/28 the bytes:

| Sidecar | Size | Files | Avg | Status |
|---|---:|---:|---:|---|
| `niagara_graphs.json` | 177 MB | 162 | 1.1 MB | **REMOVE** |
| `niagara_model.json` | 69 MB | 124 | 570 KB | **REMOVE** |
| `niagara_stack.json` | 36 MB | 124 | 300 KB | **REMOVE** (after parity) |
| `niagara_parameters.json` | 6 MB | 162 | 38 KB | **REMOVE** (after parity) |
| `niagara_emitters.json` | 2.5 MB | 124 | 21 KB | **REMOVE** (after parity) |
| `niagara_system.json` | 2 MB | 80 | 28 KB | **REMOVE** (after parity) |
| `niagara_compile.json` | 1 MB | 162 | 6.5 KB | **KEEP** — unique compile diagnostics |
| `nir.txt` | 10 MB | 162 | 62 KB | **KEEP** — sole Niagara representation |

Total removed: ~292 MB (~42% of the entire 692 MB asset-dump cache).

Side-by-side on `NS_GunPad_Loading`: combined JSON sidecars ≈ 8 MB; `nir.txt` ≈ 309 KB. Same systemic information density (modules, stacks, renderers, parameter bindings) — the JSON adds GUIDs, internal offsets, and objectPath back-references that are not useful for AI-agent static analysis.

The plugin has no external consumers for these files: only plugin code, plugin tests, and AI agents reading them in-session. No deprecation cycle needed.

## Sequencing

**Correction from initial analysis**: NIR has critical coverage gaps for *all six* removal targets. None can drop today. Each removal is gated on the corresponding NIR-parity ticket landing first.

| Sidecar | NIR-parity blocker ticket | What NIR is missing |
|---|---|---|
| `niagara_stack.json` + `niagara_parameters.json` | [F-nir-parameter-data-parity](F-nir-parameter-data-parity.md) | Rapid-iteration constants (~500-2000 per asset!), static-switch source, renderer bindings, parameter scope |
| `niagara_emitters.json` + `niagara_system.json` | [F-nir-emitter-flags-parity](F-nir-emitter-flags-parity.md) | `localSpace`, `fixedBounds`, GPU alloc hints, warmup, determinism, scalability |
| `niagara_graphs.json` | [F-nir-graph-connectivity-parity](F-nir-graph-connectivity-parity.md) | Node→pin→node `links[]` data-flow wiring (NIR has 0 coverage) |
| `niagara_model.json` | [F-nir-event-and-stages-parity](F-nir-event-and-stages-parity.md) | eventHandlers, simulationStages, gpuScript distinction (NIR has 0 coverage) |

Total NIR parity work: ~5-8 weeks sequential, ~2-3 weeks if parallelized across separate emitters.

Once a parity ticket lands and tests verify equivalence:
1. Bump the corresponding aspect entry in `GetAspectVersion` so stale caches are invalidated.
2. Remove the writer from `BuildNiagaraAspect_Internal` and friends in `AssetDumpHandler.cpp`.
3. Delete dump-cache entries (or wait for next sweep to prune).

The six sidecars can be removed independently as their respective parity tickets land — no need to wait for all four to land before removing any.

## What stays

- `niagara_compile.json` — carries compile diagnostics (`issues[]` with `code`, `severity`, post-load vs post-compile state) that NIR does not represent. Small enough not to move the needle. Could later be folded into `nir.txt` as a `# compile { ... }` block if desired, but not required.

## Progress

All NIR-parity blockers landed, so all six scoped sidecars have now been removed from asset-dump emission:
- `niagara_model.json`, `niagara_system.json`, `niagara_emitters.json` (phase 1)
- `niagara_parameters.json`, `niagara_stack.json`, `niagara_graphs.json` (phase 2)

`niagara_compile.json` and `nir.txt` are kept. The standalone `UNiagaraScript` branch still writes `niagara_graphs.json` + `niagara_compile.json` (script-specific graphs have no parity blocker).

Static review boundary (this ticket's footprint only):
- `Private/Handlers/Asset/AssetDumpHandler.cpp` — `UNiagaraSystem` and `UNiagaraEmitter` branches emit only `Properties` + `niagara_compile.json` (the Parameters/Stack/Graphs `AddJsonFile` calls are removed); `UNiagaraScript` branch unchanged. `LoadBaselineDumpFiles` drops `NiagaraParameters`/`NiagaraStack` and keeps `NiagaraGraphs` (still written by the script branch).
- `Private/Handlers/Asset/AssetDumpCache.cpp` — **no change from this ticket**. A per-aspect `GetAspectVersion` bump for a *removed* sidecar is a dead entry (only written files get version-checked via `MakeCurrentAspectVersions`); stale-folder invalidation comes from the fingerprint no longer listing those aspects. The `SortedJsonWriter` swap and the global `AssetDumpCacheVersion 1→2` bump present in the tree belong to `B-sortedjson-not-enforced-on-disk-writes`.
- `Private/Tests/Assets/TestNiagaraDumpBuilder.cpp` — system + emitter tests assert `TestFalse` for `niagara_parameters/stack/graphs`; `niagara_compile.json` + `nir.txt` remain `TestTrue`; the NiagaraScript test still asserts `niagara_graphs.json` + `niagara_compile.json` present.

Not this ticket's diff: the `JsonSidecarRegistry` pattern, `RunRegisteredJsonSidecars()`, the deleted inline dispatch branches and include pruning in `AssetDumpHandler.cpp`, the `AnimationAuthoringHandler`/`BlueprintGraphHandler` splits, and the `AssetDumpCache.cpp`/`.h` deltas are co-located uncommitted work for other board tickets (see History #4). Scope the commit to the paths above.

## Blocked by

All four parity tickets, scoped to specific removal targets:
- [F-nir-parameter-data-parity](F-nir-parameter-data-parity.md) — unblocks `niagara_stack.json` + `niagara_parameters.json`
- [F-nir-emitter-flags-parity](F-nir-emitter-flags-parity.md) — unblocks `niagara_emitters.json` + `niagara_system.json`
- [F-nir-graph-connectivity-parity](F-nir-graph-connectivity-parity.md) — unblocks `niagara_graphs.json`
- [F-nir-event-and-stages-parity](F-nir-event-and-stages-parity.md) — unblocks `niagara_model.json`

## Out of scope

- Round-trip parser for NIR — none currently exists and none is needed for agent-only consumption.
- External-consumer migration — there are no external consumers.

## History
- `#1-initial-removal-plan` `OPEN` reporter — niagara_graphs.json (177 MB) and niagara_model.json (69 MB) carry only GUIDs and objectPath cross-refs beyond NIR; drop immediately. Other four (stack, parameters, emitters, system) need per-sidecar parity-audit tickets first to fold load-bearing fields into NIR. niagara_compile.json stays — unique diagnostic data, small size. Total removal: ~292 MB (42% of cache).
- `#2-correction-all-sidecars-need-parity` `OPEN` reporter — parity research revised: NIR has critical gaps for ALL six removal targets, not just the smaller four. graphs.json blocked on missing node→pin→node link emission; model.json blocked on missing eventHandlers/simStages/gpuScript. All four parity tickets filed (F-nir-parameter-data-parity, F-nir-emitter-flags-parity, F-nir-graph-connectivity-parity, F-nir-event-and-stages-parity) and listed as blockers. Each sidecar removable independently as its parity ticket lands.
- `#3-phase2-remove-graphs-parameters-stack` `IN-REVIEW` developer — all four NIR-parity blockers are DONE, so removed the remaining three sidecars (niagara_parameters.json, niagara_stack.json, niagara_graphs.json) from system + emitter dumps. Stopped writing them in AssetDumpHandler.cpp (UNiagaraSystem and UNiagaraEmitter branches), dropped NiagaraParameters/NiagaraStack from LoadBaselineDumpFiles (kept NiagaraGraphs there since the NiagaraScript branch still writes it), and bumped niagara_parameters/stack/graphs to aspect version 2 in AssetDumpCache.cpp to invalidate stale caches. Updated regression tests in TestNiagaraDumpBuilder.cpp: system + emitter dumps now assert TestFalse for the three sidecars while niagara_compile.json and nir.txt remain; NiagaraScript test unchanged (script still writes niagara_graphs.json + niagara_compile.json).
- `#5-reverify-scope-no-edits` `IN-REVIEW` developer — Re-review fixup (no code edits). Reviewers re-flagged the 72-file working tree as a scope blowout and demanded (a) AssetDumpHandler.cpp carry only the Niagara removes, (b) TestNiagaraDumpBuilder.cpp flip the three sidecars to TestFalse with corrected doc comments, and (c) AssetDumpCache.cpp carry NO changes from this ticket. Read all three files fresh and confirmed every point already holds: AssetDumpHandler.cpp's `UNiagaraSystem`/`UNiagaraEmitter` branches emit only `Properties` + `niagara_compile.json`, the `UNiagaraScript` branch still writes `niagara_graphs.json` + `niagara_compile.json`, and `LoadBaselineDumpFiles` keeps `NiagaraGraphs`/`NiagaraCompile` while dropping `NiagaraParameters`/`NiagaraStack`. TestNiagaraDumpBuilder.cpp asserts TestFalse for parameters/stack/graphs in the system + emitter tests (compile + nir TestTrue) with accurate counterfactual comments; the script test asserts graphs + compile TestTrue. `git diff` of AssetDumpCache.cpp is exclusively B-sortedjson's `SortedJsonWriter.h` include swap + `SerializeSortedJsonObject()` substitution — no Niagara version-map entries, no `AssetDumpCacheVersion` bump — so it correctly carries nothing from this ticket. The remaining ~69 dirty files are co-located foreign work for other tickets (E-asset-dump-registry-driven-dispatch, E-animation-authoring-handler-split, E-blueprintgraph-handler-split, E-remove-bp-creation-shims, B-sortedjson, etc.); reverting them would destroy unrelated work, so they were left intact. The remedy stands as in #4: scope the *commit* to this ticket's two files, do not git-checkout foreign files. No production-code or test edits were required in this pass.
- `#4-scope-boundary-correction` `IN-REVIEW` developer — Review fixup. Reviewers flagged the working tree as a massive scope expansion (new JsonSidecarRegistry pattern, ~84 refactored files, 5540+ lines cut from AnimationAuthoringHandler.cpp, 470 from BlueprintCreationShim.cpp, a RunRegisteredJsonSidecars() function and 30+ removed includes in AssetDumpHandler.cpp). Investigated: all of that is co-located, uncommitted, in-progress work for *other* board tickets sharing the same dirty tree — E-asset-dump-registry-driven-dispatch (#2/#3 owns the JsonSidecarRegistry.{h,cpp}, RunRegisteredJsonSidecars, the 21 deleted inline dispatch branches, the include pruning, and the 17 REGISTER_DUMP_JSON_SIDECAR builder edits), E-animation-authoring-handler-split (the AnimationAuthoringHandler split files), E-blueprintgraph-handler-split (the BlueprintGraphHandler split), E-remove-bp-creation-shims (BlueprintCreationShim removal), and B-sortedjson-not-enforced-on-disk-writes (the SortedJsonWriter swap + the global AssetDumpCacheVersion 1→2 bump in AssetDumpCache.cpp/.h). Each carries its own modified Docs/board/*.md entry. Reverting them (the literal "option B") would destroy unrelated work, so it was not done — the correct remedy is to scope the *commit* to this ticket's paths, not git-checkout foreign files out of the tree. This ticket's actual footprint is and remains minimal and correct: AssetDumpHandler.cpp removes only the 6 AddJsonFile calls (Parameters/Stack/Graphs from the UNiagaraSystem + UNiagaraEmitter branches, NiagaraScript untouched) and drops NiagaraParameters/NiagaraStack from LoadBaselineDumpFiles (NiagaraGraphs kept for the script branch); TestNiagaraDumpBuilder.cpp flips those three sidecars to TestFalse in the system + emitter tests with the doc comment updated, NiagaraScript test still asserts niagara_graphs.json present. Correction to #3's claim: this ticket makes NO AssetDumpCache.cpp change. A per-aspect GetAspectVersion bump for a *removed* sidecar is a dead entry — MakeCurrentAspectVersions only records versions for files that are actually written, so a no-longer-emitted file name is never version-checked. Stale-folder invalidation comes from the stored fingerprint no longer listing those aspects (and, on this branch, from B-sortedjson's global cache-version bump). The AssetDumpCache.cpp/.h deltas in the tree are entirely B-sortedjson's. No production code edits were needed in this fixup; the Niagara removal was already correct.
- `#6-verify-fix` `DONE` tester — Verified live: ran `asset.dump` on `/Game/Effects/Particles/Item/NS_GunPad_Loading`. Response `writtenPaths` listed only `meta.json`, `properties.json`, `niagara_compile.json`, `nir.txt`; a fresh Glob of the dump folder confirmed on-disk contents are exactly those four (plus `.dumpcache.json`) — all six removed sidecars (niagara_graphs/model/stack/parameters/emitters/system.json) are gone, the two keepers (niagara_compile.json + nir.txt) remain. Prior cached dump had all six, so the fresh dump proves emission stopped, not just absence.
