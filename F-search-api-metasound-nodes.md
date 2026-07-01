---
id: F-search-api-metasound-nodes
title: "No search for MetaSound frontend node classes"
status: DONE
severity: Medium
category: feature
tags: [search, metasound, audio, frontend-registry, node-discovery]
---

# No search for MetaSound frontend node classes

MetaSound graphs are built from nodes registered in the **MetaSound
Frontend class registry** (`Metasound::Frontend::INodeClassRegistry`),
not from the UClass system. Each node is identified by a registered
`FMetasoundFrontendClass` keyed by class name + major.minor version +
interface signature. There is **no RPC that enumerates this registry**.

This is structurally different from every other catalog gap in the
issue board — the registry is internal to the MetaSound plugin and
isn't visible to `system.inspect.search_classes` (the proposal in
[`F-search-api-native-uclasses`](F-search-api-native-uclasses.md))
because MetaSound nodes are not UClasses. They're frontend-registry
entries with their own metadata schema.

**Status note:** Originally filed as Low priority gated on MetaSound
authoring being on the roadmap. That gate has lifted — `audio.authoring`
already exposes `create_metasound`, `add_metasound_node`,
`add_metasound_input`, `add_metasound_output`, `set_metasound_default`,
and `connect_metasound_nodes`. Without a node-class catalog, callers of
those RPCs are guessing class names blindly, which directly contributes
to crashes like
[`B-metasound-add-input-asserts-on-missing-literal`](B-metasound-add-input-asserts-on-missing-literal.md)
(missing or wrong class name → assert in MetaSound runtime). Promoted
to Medium severity.

**Stopgap (no code change):** Enumerate the common Epic-shipped
MetaSound node classes in the `wiki/audio.authoring.md` overlay
(Math.Add/Sub/Mul/Div, Filters.BiQuad/OnePole, Generators.Wave*,
Envelopes.ADSR, MIDI.*, etc.) so agents have a curated reference until
the live registry-walking RPC lands. Same pattern the wiki already uses
for asset-class names elsewhere.

**Use cases blocked (if MetaSound authoring is added):**

1. "What math / DSP nodes are available?" — Add, Multiply, Subtract,
   Crossfade, BiQuad filter, OnePole filter, WaveTableOscillator, etc.
2. "What input/output interface nodes are available?" — depends on
   which interfaces the project has declared (`MetaSoundSource`,
   `MetaSoundPatch`, custom interfaces).
3. Version-aware lookup — MetaSound nodes are versioned; a node may
   exist at version 1.0 and 1.1 with different signatures.

**Implementation surface:**

`Metasound::Frontend::ISearchEngine::Get().FindAllClasses(true)` returns
every registered `FMetasoundFrontendClass`. Each entry exposes name,
description, author, version, input vertices, output vertices, category
hierarchy, and deprecation status. A `audio.authoring.search_metasound_nodes`
RPC is a thin wrapper:

```
audio.authoring.search_metasound_nodes(
    query?: string,                  // keyword(s); omit for full enumeration
    category?: string,               // e.g. "Math", "Filters", "Generators"
    interfaceVersion?: { major: int, minor: int },  // exact-version pin
    includeDeprecated?: bool,         // default false
    limit?: number                    // default 50
) -> {
    results: [{
        className: "Metasound.Add",
        displayName: "Add",
        description: "Sum two operands.",
        author: "Epic Games",
        category: "Math",
        version: { major: 1, minor: 0 },
        deprecated: false,
        inputs: [{ name: "A", type: "Float" }, { name: "B", type: "Float" }],
        outputs: [{ name: "Out", type: "Float" }],
        score: number
    }],
    totalMatches: number
}
```

**Cross-ref:** Sibling to
[`F-search-api-niagara-modules`](F-search-api-niagara-modules.md) —
both are domain-specific catalogs that the generic UClass search can't
serve. MetaSound's catalog is even further removed because its registry
is entirely external to the UClass system.

## History
- `#1-no-metasound-catalog` `OPEN` reporter — Filed for completeness alongside the other discovery-gap tickets. MetaSound's frontend node registry (`Metasound::Frontend::INodeClassRegistry`) is invisible to UClass-based class search because MetaSound nodes are not UClasses — they are registry entries with their own versioned schema. Priority is gated on MetaSound authoring being on the agent roadmap; today the MCP mostly inspects MetaSound assets rather than authoring graph topology. Proposes `audio.authoring.search_metasound_nodes` wrapping `Metasound::Frontend::ISearchEngine::FindAllClasses` with keyword / category / version filters.
- `#2-reviewed-and-confirmed` `OPEN` tester — Verified live: no `search_metasound_nodes` (or any class-enumeration RPC) registered in `AudioAuthoringHandler.cpp` (grepped `search_metasound|search_node|search_class` — zero matches). Confirmed no duplicate ticket on board. Raised severity Low → Medium: MetaSound authoring is clearly on the roadmap (six `add_*`/`connect_*` MetaSound RPCs already shipped) and the missing catalog is a contributing factor to crashes like `B-metasound-add-input-asserts-on-missing-literal` where callers guess class names. Added a wiki-enumeration stopgap suggestion to unblock agents before the live RPC lands.
- `#3-search-handler` `IN-REVIEW` developer — Added `audio.authoring.search_metasound_nodes` in new `Private/Handlers/Audio/MetaSound/MetaSoundSearchHandler.cpp`. Wraps `Metasound::Frontend::ISearchEngine::Get().FindAllClasses(true)` with query / category / version / deprecated / limit filters and a simple exact>prefix>substring scoring. Falls back to `METASOUND_SEARCH_NOT_AVAILABLE` error on engine versions without the search engine API. Regression test `TestMetaSoundSearch.cpp` asserts the live registry contains a node class whose name matches "Add".
- `#4-verify-fix` `DONE` tester — Verified: `audio.authoring.search_metasound_nodes` wiki page is registered and `audio.authoring.search_metasound_nodes` with `{"query":"Add","limit":10}` returned MetaSound registry entries including `UE.Add.Float` with typed inputs/outputs and `totalMatches: 50`.
