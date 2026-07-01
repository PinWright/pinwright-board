---
id: E-asset-dump-userdefinedstruct-field-list
title: "UserDefinedStruct dumps lack field/property list — only 300-400 bytes of opaque data"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, user-defined-struct, sidecar, coverage]
---

# UserDefinedStruct dumps lack field/property list — only 300-400 bytes of opaque data

All 103 `UserDefinedStruct` dumps under `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/` contain only `meta.json` and a 300-400 byte `properties.json` whose useful content is just `Guid` + an opaque `EditorData_0` object pointer. The user-defined field list (variable names, types, defaults, GUIDs) — the entire reason an LLM agent would read a UserDefinedStruct dump — is missing.

Representative repro: `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/Blueprints/Data/ST_MASComp/properties.json` is 346 bytes total and reads:

```json
{
    "EditorData": {
        "flags": ["EditorOnly"],
        "is_overridden_locally": true,
        "type": "TObjectPtr<UObject>",
        "value": "/App/Blueprints/Data/ST_MASComp.ST_MASComp:UserDefinedStructEditorData_0"
    },
    "Guid": {
        "is_overridden_locally": true,
        "type": "FGuid",
        "value": "07F4854E41F80093E926648D821E3C9C"
    }
}
```

Two other samples (`DragDrone_Struct/properties.json`, `DragAudio_Struct/properties.json`) match exactly the same shape — only `EditorData` pointer + `Guid`. An agent reading these gets zero information about the struct's actual fields.

**Root cause.** `Handlers/Asset/AssetDumpHandler.cpp` (and the dump builders next to it: `SoundCueDumpBuilder`, `MetaSoundDumpBuilder`, `LevelSequenceDumpBuilder`, etc.) has no `UserDefinedStruct`-specific branch. The generic property walk only sees the top-level UPROPERTYs of `UUserDefinedStruct` itself (`Guid`, `EditorData`), not the runtime field list. In UE, the field metadata for a `UUserDefinedStruct` lives in `Struct->ChildProperties` (linked list of `FProperty*`) plus a `Guid` + per-field GUIDs in editor data. The plugin's `Utils/PropertyUtils` already knows how to serialize `FProperty` info, so a small new dump builder can reuse it.

**Fix:** Emit a `user_defined_struct.json` sidecar (matching the existing `sound_cue.json` / `metasound.json` / `level_sequence.json` idiom — registered around `AssetDumpHandler.cpp` lines 530/703) with:

```
{
  "fields": [
    { "name": "...", "displayName": "...", "type": "...", "subType": "...", "defaultValue": ..., "guid": "..." },
    ...
  ]
}
```

Walk `Struct->ChildProperties`; pull display name + per-field GUID from the user-defined struct's editor data; reuse `PropertyUtils` for type/subtype/default serialization.

## History
- `#1-initial-repro` `OPEN` reporter — 103 `UserDefinedStruct` dumps under `.editor-automation/asset-dumps/` are effectively empty: only `meta.json` + a 300-400 byte `properties.json` containing just `Guid` + an opaque `EditorData_0` object pointer (sample: `App/Blueprints/Data/ST_MASComp/properties.json`, 346 bytes; same shape on `DragDrone_Struct`, `DragAudio_Struct`). The user-defined field list (names, types, defaults, GUIDs) is missing. `Handlers/Asset/AssetDumpHandler.cpp` and the neighboring dump builders have no UDS-specific branch, so the generic property walk only sees the top-level UPROPERTYs of `UUserDefinedStruct` itself. Proposed fix: add a `user_defined_struct.json` sidecar (matching the `sound_cue.json` / `metasound.json` idiom) that walks `Struct->ChildProperties` and reuses `Utils/PropertyUtils` for type info.
- `#2-add-uds-sidecar` `IN-REVIEW` developer — Added `UserDefinedStructDumpBuilder.{h,cpp}` walking `FStructureEditorUtils::GetVarDesc(Struct)` and emitting `user_defined_struct.json` (assetKind/path/guid/fields[name/displayName/guid/type/subType/subTypeObject/containerType/defaultValue/currentDefaultValue/tooltip/flags/metaData]). Wired dispatch in `AssetDumpHandler.cpp::BuildAllFilesForAsset` after the `UDataTable` branch; added `DumpFileNames::UserDefinedStruct` constant and registered in `Canonical[]`. Regression test `TestUserDefinedStructDumpBuilder.cpp` with `FUserDefinedStructDumpBuilderShapeTest` (unit, transient UDS with two variables) and `FUserDefinedStructAssetDumpWritesUDSAspectFileTest` (end-to-end via `DumpSingleAsset`). Counterfactual: reverting the dispatch branch makes the end-to-end test fail because `HasDumpFile(Result.WrittenPaths, DumpFileNames::UserDefinedStruct)` returns false.
- `#3-verify-fix` `DONE` tester — Verified: `asset.dump` on `/App/Blueprints/Data/ST_MASComp` now writes `user_defined_struct.json` alongside `meta.json` + `properties.json` (Files (3) in dump response). Sidecar contains assetKind=UserDefinedStruct, guid=07F4854E41F80093E926648D821E3C9C, and a 7-element `fields` array with full schema (name/displayName/guid/type/subType/subTypeObject/containerType/defaultValue/currentDefaultValue/tooltip/flags/metaData) matching the IN-REVIEW contract — e.g. `Mesh` field reports type=object, subTypeObject=/Script/Engine.StaticMesh.
