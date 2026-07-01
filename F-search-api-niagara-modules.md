---
id: F-search-api-niagara-modules
title: "No search for Niagara module scripts by usage / context"
status: DONE
severity: Medium
category: feature
tags: [search, niagara, module-discovery, asset-registry, stack]
---

# No search for Niagara module scripts by usage / context

Niagara system / emitter stacks are built by attaching **module scripts**
(`UNiagaraScript` assets with `Usage` ∈ `Module`, `DynamicInput`,
`Function`, …) at specific stack groups (`ParticleSpawn`,
`ParticleUpdate`, `EmitterUpdate`, `SystemUpdate`, …). `niagara.add_module`
takes a `scriptAssetPath` and the target stack location, but there is
**no RPC that enumerates which module scripts are valid for a given
stack location**.

The catalog isn't a UClass catalog (the script class is always
`UNiagaraScript`) — it's an **asset-registry search filtered by the
script's `Usage` enum and module bitmask**. `asset.search` /
`asset.search_assets` don't model Niagara-specific usage filters, so the
caller can't ask "give me every script asset eligible for the
ParticleUpdate stack group".

**Use cases blocked:**

1. "What module scripts can I add to ParticleUpdate?" — caller wants a
   ranked list of `UNiagaraScript` assets where `Usage == Module` and the
   script's allowed-stage bitmask includes `ParticleUpdate`. Today: read
   the editor's stack-add menu manually, or `python.execute` walking the
   asset registry plus per-script usage validation.
2. Dynamic-input discovery — for `niagara.set_module_input` with a
   dynamic-input override, the caller needs to know which
   `Usage == DynamicInput` scripts produce the right type. Same gap.
3. Built-in vs project module differentiation — `/Niagara/Modules/...`
   (engine plugin) vs `/Plugin/<X>/Modules/...` (game-feature plugin) vs
   `/Game/...` (project content) are all valid sources; the caller often
   wants to prefer one or filter by source.

**Current workarounds:**

- `python.execute` calling `unreal.AssetRegistryHelpers.get_asset_registry()`
  and filtering by `UNiagaraScript::GetUsage()`.
- `asset.search query="*" parentClassPath="/Script/Niagara.NiagaraScript"`
  returns *every* script regardless of usage, then the caller must load
  each and inspect its usage — O(N) editor stalls.
- Read the editor's Niagara stack "add module" dropdown manually.

All bypass the typed-RPC surface.

**Proposal:** Add `niagara.search_modules` (and `niagara.list_modules`):

```
niagara.search_modules(
    query?: string,               // keyword(s) on asset name + description; omit for unranked enumeration
    usage: "Module"|"DynamicInput"|"Function"|"Particle*"|"Emitter*"|"System*",  // required — narrows the registry walk
    stage?: "ParticleSpawn"|"ParticleUpdate"|"EmitterSpawn"|"EmitterUpdate"|"SystemSpawn"|"SystemUpdate",  // optional further filter
    inputType?: string,           // for DynamicInput: required output type (e.g. "Niagara.Vector", "Niagara.Float")
    sourceFilter?: "engine"|"plugin"|"project"|"any",  // default "any"
    limit?: number                // default 50
) -> {
    results: [{
        assetPath: "/Niagara/Modules/Forces/CurlNoiseForce.CurlNoiseForce",
        name: "Curl Noise Force",
        description: "Adds curl noise as a force to particle velocity",
        usage: "Module",
        validStages: ["ParticleUpdate", "ParticleSpawn"],
        inputs: [{ name: "NoiseStrength", type: "Niagara.Float" }, ...],
        outputs: [],                  // modules write to dataset, dynamic inputs return a value
        source: "engine",             // engine / plugin / project
        score: number
    }],
    totalMatches: number
}
```

Implementation surface: walk the asset registry filtered to
`/Script/Niagara.NiagaraScript`, load each (lazy / on-demand if perf
matters), read `UNiagaraScript::Usage` + `ModuleUsageBitmask` +
`bDeprecated` + signature. Cache the result keyed by usage so repeated
calls in a session don't re-walk. Optionally back this with a one-time
`niagara.build_module_index` call mirroring `blueprint.build_api_index`
if cold cost is high.

**Cross-ref:** Adjacent to
[`F-search-api-native-uclasses`](F-search-api-native-uclasses.md) but
solves a different problem — Niagara's catalog is *asset-registry*, not
*UClass*. Both are needed.

**Workflow gotcha to surface in the result:** a freshly-created emitter
stack is empty; adding the wrong-usage script silently no-ops or attaches
to the wrong group. The wiki already calls this out for `niagara.*` edit
RPCs — exposing the stage validity in the search result avoids the
guess-and-check cycle.

## History
- `#1-no-niagara-module-catalog` `OPEN` reporter — `niagara.add_module` takes `scriptAssetPath` but provides no discovery surface for which scripts are eligible. The catalog is asset-registry-shaped (not UClass), filtered on `UNiagaraScript::Usage` + `ModuleUsageBitmask`, and `asset.search` doesn't model these filters. Today the agent must either know the script path already or walk every `UNiagaraScript` via Python. Proposes `niagara.search_modules` with `usage` (required), `stage`, `inputType`, and `sourceFilter` parameters, returning asset path + validity stages + input/output signature so the result is directly consumable by `niagara.add_module` / `niagara.set_module_input`. Complements [`F-search-api-niagara-graph-nodes`](F-search-api-niagara-graph-nodes.md) which addresses raw `UNiagaraNode*` graph-node discovery.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Verified by source-reading the Niagara handler directory (`Source/Handlers/Niagara/`): grepped every `REGISTER_RPC_HANDLER("niagara.*")` — 28 RPCs covering add/remove/move/inspect/set on emitters, modules, parameters, renderers, data interfaces, event handlers, sim stages, plus `niagara.graph.*` low-level mutations and `niagara.inspect`/`niagara.validate`/`niagara.decompile_model`. None enumerate the script-asset catalog. Confirmed `asset.search_assets` filters by `FARFilter` (class paths, package paths, recursion only) — no per-asset-type usage/bitmask hook, so `parentClassPath="/Script/Niagara.NiagaraScript"` returns every script regardless of `Usage`, exactly as the ticket states. The "extend `asset.search_assets` with a `niagaraUsage` filter instead of a new RPC" alternative was considered and rejected: `Usage` and `ModuleUsageBitmask` aren't asset-registry tags by default (would require Niagara-side `GetAssetRegistryTags` work or per-asset loads regardless), and the result shape needs Niagara-shaped fields (`validStages`, `inputs[]`, signature) that don't fit the generic `asset.search_assets` envelope — a typed `niagara.search_modules` RPC keeps the surface coherent with the existing `niagara.*` family. Cross-ref with `F-search-api-niagara-graph-nodes` is legitimately separate: that ticket covers the `UNiagaraNode*` taxonomy + op registry inside script graphs (module-authoring concern, Low priority); this one covers the stack-module asset catalog (stack-composition concern, Medium priority). Severity Medium is appropriate — caller can fall back to `python.execute` walking the registry, but it stalls the editor per-asset. No duplicate file in the board folder. Ticket stands as filed.
- `#3-implemented-search-modules` `IN-REVIEW` developer — Added `niagara.search_modules` to `NiagaraSearchHandler.cpp` (shared with the graph-nodes search ticket). Filters via FAssetData tags (`Usage` reflection-emitted, `ModuleUsageBitmask` explicit tag) so the common case is load-free; `inputs[]`/`outputs[]` signature emission deferred to a follow-up (requires per-asset load). Source classification (engine/plugin/project) and scored ranking via helpers exposed in NiagaraSearchHandler.h.
- `#4-fix-review-issues` `IN-REVIEW` developer — Fixed `ClassifySource` to use `StartsWith("/Niagara/")` instead of `Contains` (prevents `/Game/Niagara/Foo` false-positive as "engine"); cached `Description`, `Keywords`, and `Source` in `FScoredAsset` to eliminate duplicate `GetTagValue`/`ClassifySource` calls per result; added false-positive test case to `TestNiagaraSearchModules.cpp`; updated header doc comment.
- `#5-returned-stage-alias-missing` `OPEN` tester — Returned: `niagara.search_modules` rejects the documented stack stage value `ParticleUpdate` with `INVALID_STAGE`, while `ParticleUpdateScript` works and returns engine module records with `usage: "Module"` and `source: "engine"`. Test: `niagara.search_modules` with `{"usage":"Module","stage":"ParticleUpdate","sourceFilter":"engine","query":"Curl Noise Force","limit":5}`.
- `#6-normalize-stage-aliases` `IN-REVIEW` developer — Updated `niagara.search_modules` stage decoding in `NiagaraSearchHandler.cpp` to accept documented stack-stage aliases such as `ParticleUpdate` while preserving `*Script` enum-name aliases, normalized `validStages` output to the documented alias form, and added `FNiagaraSearchModulesStageAliasTest` to cover the returned `ParticleUpdate` case.
- `#7-verify-stage-alias` `DONE` tester — Verified: `niagara.search_modules` with `{"usage":"Module","stage":"ParticleUpdate","sourceFilter":"engine","limit":5}` returned 201 engine modules (e.g. `/Niagara/Modules/Update/Forces/AccelerationForce`), no `INVALID_STAGE`. `stage:"ParticleUpdateScript"` enum-name alias still accepted (no error). `validStages` output normalized to documented form (`ParticleSpawn`, `ParticleUpdate`, `ParticleSimulationStage`, …), not `*Script` suffixes. The #5 returned symptom is resolved.
