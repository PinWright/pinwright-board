---
id: B-asset-dump-niagarascript-no-graph-sidecar
title: "asset.dump standalone NiagaraScript / NiagaraModule assets get no niagara_*.json sidecar"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, niagara, graph]
---

# asset.dump standalone NiagaraScript / NiagaraModule assets get no niagara_*.json sidecar

NiagaraScript assets (the most reusable Niagara unit) have only `meta.json` + `properties.json`. The rich Niagara sidecar set (`niagara_graphs.json`, `niagara_compile.json`, etc.) only triggers for NiagaraSystem / NiagaraEmitter dispatch.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Effects/NiagaraModules/NM_BPSystemEvent/`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Effects/NiagaraModules/NM_BPSystemEventSpawnMask/`.
3. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Effects/NiagaraModules/NM_FootPrintSizeAndOrientation/`.
4. Observe: no `niagara_graphs.json` / `niagara_compile.json` — only meta.json + properties.json. 14/14 NiagaraScript assets in slice.

**Fix (proposed):** `AssetDumpHandler` emits niagara_*.json under `Cast<UNiagaraSystem>` / `Cast<UNiagaraEmitter>` branches. Add a `Cast<UNiagaraScript>` branch that emits at least the script's graph + compile state, mirroring the system's sidecar shape.

## History
- `#1-initial-repro` `OPEN` reporter — Standalone NiagaraScript / NiagaraModule assets get only meta.json + properties.json; the niagara_*.json sidecar set fires only on NiagaraSystem / NiagaraEmitter. Sample paths: `Game/Effects/NiagaraModules/NM_BPSystemEvent/`, `Game/Effects/NiagaraModules/NM_BPSystemEventSpawnMask/`, `Game/Effects/NiagaraModules/NM_FootPrintSizeAndOrientation/`. 14/14 NiagaraScript assets observed.
- `#2-niagarascript-sidecars` `IN-REVIEW` developer — Added standalone UNiagaraScript graph/compile dump sidecars through NiagaraDumpBuilder and AssetDumpHandler, with regression coverage for niagara_graphs.json and niagara_compile.json.
- `#3-verify-fix` `DONE` tester — Verified: asset.dump on /Game/Effects/NiagaraModules/NM_BPSystemEvent and /Game/Effects/NiagaraModules/NM_FootPrintSizeAndOrientation both emit niagara_graphs.json (2635 lines) and niagara_compile.json alongside meta.json/properties.json.
