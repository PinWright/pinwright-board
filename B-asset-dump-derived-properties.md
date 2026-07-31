---
id: B-asset-dump-derived-properties
title: "properties.json leaks derived engine state and produces false asset diffs"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, properties, determinism, derived-data]
---

# `properties.json` leaks derived engine state and produces false asset diffs

An unchanged `/App` `asset.dump_folder` sweep changed 16 `properties.json` files solely because generic reflection exports engine-owned derived state: 12 StaticMesh dumps changed `FStaticMeshSourceModel::CacheMeshDescription*Count` between built counts and `MAX_uint32`; a material's `TextureStreamingData` reordered/resolved; two LevelSequence signatures regenerated; and a NiagaraSystem gained `ScriptRuntimeCompiledDataForEditor` plus changed `SystemCompiledData`.

The exporter filters only known texture/oversized fields and top-level `CPF_Transient | CPF_DuplicateTransient` properties (`PropertyExport.cpp:1355-1375`). Nested structs are walked without any property-flag or semantic filter (`PropertyExport.cpp:443-451`), so persisted caches remain visible. This is not fixed by `B-asset-dump-property-unsupported-sentinel-on-common-types` (DONE: top-level transient skip and typed encoding) or the broader `B-asset-dump-tier3-determinism-backlog` (OPEN: DDC summary stats/TSet ordering).

Engine declarations confirm these are not authored state: `StaticMeshSourceData.h:167-171` defaults both cache counts to `MAX_uint32`; `MovieSceneSignedObject.h:83-93` says `Signature` regenerates on change and equivalent state does not produce the same GUID; `MaterialInterface.h:420-422` calls `TextureStreamingData` texture-streaming data and `MaterialInterface.cpp:2350-2405` mutates/sorts it; `NiagaraSystem.h:1002-1011` labels both fields post-compile/runtime compiled data.

**Workaround:** manually ignore these owner/property pairs when reviewing dump diffs.
**Fix:** add a centralized owner-type + property-name exclusion for non-semantic derived fields, apply transient/duplicate/deprecated filtering inside `StructToJsonObject`, and pin each exclusion with a regression test. Bump the `properties.json` aspect version.

## History
- `#1-app-redump-derived-noise` `OPEN` reporter — `asset.dump_folder {folderPath:"/App",recursive:true}` completed, but the Git diff contained 16 derived-only `properties.json` changes: 12 StaticMesh cached-count flips, two regenerated MovieScene signatures, one reordered material streaming cache, and one Niagara compiled-data change. Source and UE 5.8 declarations confirm these fields are runtime/derived rather than authored asset state.
- `#2-filter-nonsemantic-properties` `IN-REVIEW` developer — Property export now omits transient, duplicate-transient, deprecated, and skip-serialization fields at nested and top-level boundaries, plus exact owner-qualified derived caches for StaticMesh source counts, MovieScene signatures, material texture-streaming data, and Niagara compiled data. Set values are sorted by canonical JSON. Two cold forced 8,192-asset `/App` sweeps produced the same diff hash; the one-time cleanup removed repeated collision, physics, and editor caches rather than carrying session state forward.
