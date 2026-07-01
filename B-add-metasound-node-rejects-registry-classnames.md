---
id: B-add-metasound-node-rejects-registry-classnames
title: "add_metasound_node rejects every className search_metasound_nodes advertises; nodeType shorthands map to stale Metasound.* names"
status: IN-REVIEW
severity: High
category: bug
tags: [metasound, audio, authoring, node-registry, classname, add_metasound_node, search_metasound_nodes]
---

# add_metasound_node cannot consume the className its sibling search RPC reports

`audio.authoring.search_metasound_nodes` and `audio.authoring.add_metasound_node`
disagree on the MetaSound Frontend className namespace, so the documented
"search the registry, then add the node" workflow is broken end to end. There
is **no className spelling** that `add_metasound_node` accepts for a node that
`search_metasound_nodes` reports as existing, and the `nodeType` shorthand
mappings point at class names that are not in the live registry either.

This leaves `add_metasound_node` with **zero working node-add path** for the
shipped Epic generator/math nodes (Sine, Multiply, etc.). A caller can build a
MetaSound asset with inputs, outputs, and baked-in defaults, but can never add a
single DSP node to wire a signal path — the saved asset is silent.

## What's wrong

1. **search/add className mismatch.** The search RPC returns the registry key
   `UE.Sine.Audio` (version 1.1, category Generators, non-deprecated). Feeding
   that exact string straight back into `add_metasound_node` as `nodeClassName`
   returns `NODE_CLASS_NOT_FOUND`. The output of one sibling RPC is not a valid
   input to its sibling — the two speak different className dialects (`UE.*`
   registry keys vs. whatever the add resolver expects).

2. **Stale `nodeType` shorthands.** `AudioAuthoringHandler.cpp:752-755` hardcodes
   the shorthand map: `oscillator`/`sine` -> `Metasound.Sine`,
   `gain`/`multiply` -> `Metasound.Multiply`. Neither `Metasound.Sine` nor
   `Metasound.Multiply` exists in the current registry (the real sine node is
   `UE.Sine.Audio`; the search ticket `F-search-api-metasound-nodes` `#4` itself
   verified the add node is `UE.Add.Float` — i.e. the `UE.*` namespace, not
   `Metasound.*`). So the documented shorthand `nodeType: "oscillator"` also
   fails with `NODE_CLASS_NOT_FOUND` for `Metasound.Sine`.

3. **No version disambiguation.** The registry distinguishes nodes by
   major.minor version, and `search_metasound_nodes` reports it, but
   `add_metasound_node` has no `versionMajor`/`versionMinor` params
   (`UNKNOWN_PARAMS` when passed), so even a correct base className can't be
   pinned to a version.

## What it should do

`add_metasound_node` should accept the canonical registry className that
`search_metasound_nodes` returns (`UE.Sine.Audio`, `UE.Add.Float`, ...) and
resolve it against the same `Metasound::Frontend` registry the search RPC walks
(`ISearchEngine::FindAllClasses`). The `nodeType` shorthand table should map to
class names that actually exist on the running engine (or be derived from the
registry rather than hardcoded). Search output -> add input must round-trip.

## Verbatim repro

1. `audio.authoring.create_metasound` `{name:"MS_ReplayBlip", path:"/Game/Audio"}`
   -> ok, `assetPath:/Game/Audio/MS_ReplayBlip`.
2. `audio.authoring.search_metasound_nodes` `{query:"Sine", limit:10}`
   -> first result verbatim: `{"className":"UE.Sine.Audio","displayName":"Sine",`
   `"category":"Generators","version":{"major":1,"minor":1},"deprecated":false,...}`.
3. `audio.authoring.add_metasound_node`
   `{assetPath:"/Game/Audio/MS_ReplayBlip", nodeClassName:"UE.Sine.Audio"}`
   -> `[NODE_CLASS_NOT_FOUND] Node class 'UE.Sine.Audio' not found in MetaSound registry`.
4. `audio.authoring.add_metasound_node`
   `{assetPath:"/Game/Audio/MS_ReplayBlip", nodeType:"oscillator"}`
   -> `[NODE_CLASS_NOT_FOUND] Node class 'Metasound.Sine' not found in MetaSound registry`
   (shorthand resolves to the stale `Metasound.Sine`).
5. `audio.authoring.add_metasound_node`
   `{..., nodeClassName:"UE.Sine.Audio", versionMajor:1, versionMinor:1}`
   -> `[UNKNOWN_PARAMS] ... Valid parameters: [assetPath, nodeClassName, nodeType, save]`
   (no way to pin a version).

## Impact

Blocks all MetaSound DSP graph authoring via the MCP. `create_metasound`,
`add_metasound_input`, `add_metasound_output`, and `set_metasound_default` all
work and round-trip, but with no addable node the graph has no signal path. The
asset validates "clean" and saves successfully while being functionally silent,
so a caller can ship a broken asset believing the workflow succeeded.

**Workaround:** none from the MCP surface — every documented and registry-derived
className form is rejected.

**Fix:** route `add_metasound_node`'s className resolution through the same
`Metasound::Frontend` registry/search engine that `search_metasound_nodes`
uses so the search RPC's `className` values resolve directly; refresh the
`nodeType` shorthand table to current `UE.*` class names (or look them up in the
registry by display name/category); add optional `versionMajor`/`versionMinor`
to disambiguate versioned nodes.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live against `mcp__editor-automation__call`. `search_metasound_nodes {query:"Sine"}` returns `className:"UE.Sine.Audio"`; `add_metasound_node {nodeClassName:"UE.Sine.Audio"}` returns `[NODE_CLASS_NOT_FOUND]` for that exact verbatim string; `nodeType:"oscillator"` fails because the shorthand at `AudioAuthoringHandler.cpp:752-755` maps to the stale `Metasound.Sine`/`Metasound.Multiply` (not in registry); `versionMajor`/`versionMinor` rejected as `UNKNOWN_PARAMS`. Root cause grounded in source: the add resolver and the search engine use different className namespaces (`Metasound.*` shorthand vs. live `UE.*` registry keys). No DSP node can be added by any spelling. Deduped against `F-search-api-metasound-nodes` (DONE — adds search but doesn't cover its output being non-consumable by add) and `B-metasound-add-input-asserts-on-missing-literal` (different RPC/crash); genuinely new.
- `#2-fix` `IN-REVIEW` developer — Root cause was narrower than "different registry": `add_metasound_node` used the SAME registry but fed it an unparsed flat string. Fixed in `AudioAuthoringHandler.cpp`: (1) replaced the malformed `FMetasoundFrontendClassName(FName(), FName(*ActualClassName), FName())` construction with `FMetasoundFrontendClassName::Parse(ActualClassName, ClassName)` (engine `MetasoundFrontendDocument.cpp:820`, splits `Namespace.Name[.Variant]`), with the old flat-name form kept only as a no-dot fallback — so search output (`UE.Sine.Audio`) now round-trips into add; (2) refreshed the stale `nodeType` shorthands from `Metasound.*` to live `UE.*` keys verified against the engine: oscillator/sine→`UE.Sine.Audio`, gain/multiply→`UE.Multiply.Audio by Float`, add→`UE.Add.Float`, waveplayer→`UE.Wave Player.Mono` (namespace `UE` per `MetasoundStandardNodesNames.cpp:10`/`MetasoundEngineNodesNames.cpp:9`); (3) added optional `versionMajor`/`versionMinor` params (clears the `UNKNOWN_PARAMS` repro) — `versionMajor` drives the public `AddNodeByClassName(ClassName, Major, …)` overload which resolves the highest registered minor (the major+minor overload is `UE_INTERNAL`/deprecated, deliberately not called to avoid a deprecation-as-error build risk); the response now reports both the registry-resolved `nodeClassName` and the `requestedClassName`. Regression test `EditorAutomationRpcGateway.Assets.AddMetaSoundNodeResolvesRegistryClassName` added to `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestMetaSoundDestructiveOps.cpp`: seeds a transient MetaSound, probes the registry (skips only if MetaSound truly uninitialized), then invokes the production handler asserting `UE.Sine.Audio` resolves and round-trips, the `oscillator` shorthand resolves, and `versionMajor`/`versionMinor` are no longer `UNKNOWN_PARAMS`. The test fails if either the Parse fix or the shorthand refresh is reverted. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`, `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestMetaSoundDestructiveOps.cpp`. Not compiled/tested here (later phase).
