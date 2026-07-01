---
id: F-rpc-audio-describe-metasound
title: "Add live RPC `audio.authoring.describe_metasound` for full MetaSound graph"
status: DONE
severity: High
category: feature
tags: [audio, metasound, asset-dump, parity]
---

# Add live RPC `audio.authoring.describe_metasound` for full MetaSound graph

`MetaSoundDumpBuilder::BuildMetaSoundJson` (Source/EditorAutomationRpcGateway/Private/Handlers/Asset/MetaSoundDumpBuilder.cpp) emits a structured snapshot of any `UMetaSoundSource` / `UMetaSoundPatch` — `assetKind`, `assetPath`, `rootGraph` (classID/className/isPreset + interface inputs/outputs with default literals), full `nodes` (id, classID, name, per-vertex inputs/outputs with literal lookup), `edges` (fromNodeID/fromVertexID → toNodeID/toVertexID), and `variables`. This is reachable only through `asset.dump` / `asset.dump_folder` writing `metasound.json` to the on-disk cache.

The live summary, `audio.authoring.get_audio_info` (AudioAuthoringHandler.cpp), branches on SoundCue / SoundWave / SoundClass / SoundMix / SoundAttenuation. `UMetaSoundSource` is caught by the SoundWave branch because it derives from `USoundWaveProcedural`, while `UMetaSoundPatch` falls through to `type: "Unknown"`. Neither path consults `IMetaSoundDocumentInterface`, so neither exposes the MetaSound graph. All other entries in the `audio.authoring.*` MetaSound surface (`create_metasound`, `add_metasound_node`, `add_metasound_input`, `add_metasound_output`, `connect_metasound_nodes`, `set_metasound_default`) are mutators. There is no live read path at all for MetaSound graphs.

This violates the policy that asset-dump output must not be exclusive — anything emitted to a sidecar must also be reachable via a live RPC. Today an agent that wants the graph either has to run `asset.dump` and read the file, or fall back to the legacy 41 KB single-line `RootMetasoundDocument` ExportText blob (the very thing `B-asset-dump-metasound-no-graph-builder` was filed to remove).

**Fix:** Add `audio.authoring.describe_metasound` (alongside `get_audio_info`) accepting `assetPath` and returning the JSON object produced by `MetaSoundDumpBuilder::BuildMetaSoundJson(Asset)` directly — the builder already accepts `UObject*` and downcasts via `IMetaSoundDocumentInterface`, and the symbol is `EDITORAUTOMATIONRPCGATEWAY_API`-exported, so the handler is a thin wrapper: load the asset, hand it to the builder, send the result. Error with `ASSET_NOT_FOUND` when load fails and `INVALID_TYPE` when the cast to `IMetaSoundDocumentInterface` returns nullptr (mirror the dump path's nullptr behaviour). Keep `get_audio_info` as the lightweight summary surface; describe_metasound is the structural-graph surface, parallel to how `niagara.*` exposes both summary and graph reads.

## History
- `#1-initial-repro` `OPEN` reporter — `audio.authoring.get_audio_info` falls through to `type: "Unknown"` for `UMetaSoundSource` / `UMetaSoundPatch` (AudioAuthoringHandler.cpp:2322-2358 has no IMetaSoundDocumentInterface branch). `MetaSoundDumpBuilder::BuildMetaSoundJson` produces the full graph (rootGraph + interface + nodes + edges + variables + per-input default literals) but is reachable only via `asset.dump` writing `metasound.json`. All other `audio.authoring.*` MetaSound RPCs are writers (create/add/connect/set). Violates the rule that dump output must not be exclusive. Proposed wrapper RPC: `audio.authoring.describe_metasound { assetPath }` → returns `MetaSoundDumpBuilder::BuildMetaSoundJson(Asset)` directly.
- `#2-describe-metasound-rpc` `IN-REVIEW` developer — added read-only `audio.authoring.describe_metasound` in `AudioAuthoringHandler.cpp` using `MetaSoundDumpBuilder::BuildMetaSoundJson` after generic asset load and `IMetaSoundDocumentInterface` validation; updated `audio.authoring` wiki guidance to use the new RPC for MetaSound graph snapshots; added `FAudioAuthoringDescribeMetaSoundReturnsGraphJsonTest` covering a transient `UMetaSoundPatch` result with `assetKind`, `rootGraph`, `nodes`, `edges`, and `variables`. Also corrected the body: `UMetaSoundSource` is covered by the SoundWave branch, but neither source nor patch exposes graph JSON through `get_audio_info`.
- `#3-verify-fix` `DONE` tester — Verified: `audio.authoring.describe_metasound` wiki page exists with `assetPath` parameter; invoked with `/Game/Audio/MetaSounds/MS_PlayOneShot_2ch.MS_PlayOneShot_2ch` returned 77KB JSON containing all required top-level fields (`assetKind: "MetaSoundSource"`, `assetPath`, `rootGraph`, `nodes`, `edges`, `variables`) matching `MetaSoundDumpBuilder::BuildMetaSoundJson` schema.
