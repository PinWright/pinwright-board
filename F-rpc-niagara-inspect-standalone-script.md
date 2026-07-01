---
id: F-rpc-niagara-inspect-standalone-script
title: "Extend niagara.inspect/graph.get/validate to accept standalone NiagaraScript and NiagaraModule assets"
status: DONE
severity: Medium
category: feature
tags: [niagara, niagara-graph, asset-dump-parity, rpc]
---

# Extend niagara.inspect/graph.get/validate to accept standalone NiagaraScript and NiagaraModule assets

`asset.dump` now emits `niagara_graphs.json` and `niagara_compile.json` sidecars for standalone `UNiagaraScript` assets (NiagaraModule / NiagaraFunction / NiagaraDynamicInput) via `NiagaraDumpBuilder::BuildScriptGraphsJson` and `BuildScriptCompileJson` (see AssetDumpHandler.cpp:495-500, fix landed in B-asset-dump-niagarascript-no-graph-sidecar). The matching live RPCs do not: `niagara.inspect`, `niagara.graph.get`, and `niagara.validate` only `Cast<UNiagaraSystem>` / `Cast<UNiagaraEmitter>` and reject everything else with `UNSUPPORTED_ASSET` (NiagaraInspectHandler.cpp:230-247, NiagaraGraphHandler.cpp:199-213, NiagaraInspectHandler.cpp:296-322).

This violates the asset-dump-parity policy: anything reachable through a sidecar must also be reachable through a live RPC. Today, an agent inspecting a NiagaraModule must read the cached dump and cannot fall back to a fresh live read after edits.

**Fix:** Add a `Cast<UNiagaraScript>` branch in each of the three handlers that delegates to the existing `BuildScriptGraphsJson` / `BuildScriptCompileJson` helpers (and a property/stack equivalent if/when one is added). No new builder logic needed; just route the cast and tag `assetKind` as `NiagaraScript`. Update `Docs/wiki/niagara.md` and `Docs/wiki/niagara.graph.md` to document the new accepted asset kinds.

## History
- `#1-initial-repro` `OPEN` reporter — `asset.dump` emits niagara_graphs.json + niagara_compile.json for standalone UNiagaraScript assets via NiagaraDumpBuilder::BuildScriptGraphsJson/BuildScriptCompileJson (AssetDumpHandler.cpp:495-500), but `niagara.inspect`, `niagara.graph.get`, and `niagara.validate` reject the same assets with UNSUPPORTED_ASSET. Asset-dump-parity policy requires live RPC coverage. Fix: add a Cast<UNiagaraScript> branch in each handler that calls the existing builders.
- `#2-standalone-script-live-reads` `IN-REVIEW` developer — Updated NiagaraInspectHandler.cpp and NiagaraGraphHandler.cpp with UNiagaraScript branches using NiagaraDumpBuilder script graph/compile helpers, documented standalone script/module/function/dynamic-input live reads in docs/wiki/niagara.md and docs/wiki/niagara.graph.md, and added FNiagaraStandaloneScriptLiveReadHandlersTest regression coverage in TestNiagaraHandlers.cpp.
- `#3-verify-standalone-script-live-reads` `DONE` tester — Verified: ran niagara.inspect, niagara.graph.get, and niagara.validate against /Niagara/Modules/Update/Color/ScaleColor.ScaleColor (a standalone UNiagaraScript Module). All three returned success with `assetKind: "NiagaraScript"` — no UNSUPPORTED_ASSET. validate returned scriptUsage=Module and a populated scripts[] entry; inspect/graph.get returned multi-thousand-line graph/compile payloads.
