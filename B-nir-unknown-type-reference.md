---
id: B-nir-unknown-type-reference
title: "NIR emits 'Unknown' type for custom struct / NDI references"
status: DONE
severity: Medium
category: bug
tags: [niagara, nir, type-resolution]
---

# NIR emits 'Unknown' type for custom struct / NDI references

Graph/module NIR references a literal `Unknown` type for getter operations on custom Niagara struct types or NDIs (e.g. `UNiagaraDataInterfaceGrid3DCollection`, `UNiagaraDataInterfaceSkeletalMesh`):

```
get $Add : Unknown
```

instead of the actual type name. Affects 14+ instances in a single sampled module (`NM_ProjectToPlane`).

## Sample paths

- `Game/Niagara/Modules/Custom/NM_ProjectToPlane/nir.txt` — lines 7, 25, 35, 42, 44, 61 (multiple).
- Common in any system using non-stock NDIs.

## Fix sketch

`NIRDecompiler.cpp` type-resolution path needs to handle:
- Custom UStruct types registered via `FNiagaraTypeRegistry::Register`
- Data-interface types (UNiagaraDataInterface subclasses)
- User-defined structs from blueprints

Emit fully qualified type path (e.g. `NDI::SkeletalMesh`, `Struct::FMyCustomData`) instead of `Unknown`. If resolution truly fails, emit the underlying engine class name as a fallback before `Unknown`.

## History
- `#2-nir-parity-wave-plan` `IN-REVIEW` implementer — NIR pin type formatting now falls back to explicit struct/class/enum pin metadata before returning `Unknown`.
- `#1-unknown-type-refs` `OPEN` reporter — NIR consumers can't validate data flow through pins of these types. Affects any system using custom NDIs (skeletal mesh sampling, grids, etc.).
- `#3-still-unknown-on-add-pin` `OPEN` tester — Returned: fresh `asset.dump` on `/Game/Effects/NiagaraModules/NM_ProjectToPlane` still emits 12 instances of `get $Add : Unknown` (e.g. line 86 `%NiagaraNodeParameterMapGet_2.Add = get $Add : Unknown`, lines 104, 114, 121, 123, 140, 143, 155, 158, 174, 176, 181). Concrete struct/NDI types (NiagaraPosition, Vector3f, NiagaraDataInterfaceCamera) resolve correctly now, but the `Add`-new-pin metadata on `NiagaraNodeParameterMapGet` still falls through to `Unknown`. Test: `mcp__editor-automation__call` method=`asset.dump` args=`{"assetPath":"/Game/Effects/NiagaraModules/NM_ProjectToPlane"}`, then grep `nir.txt` for `: Unknown`.
- `#4-add-pin-suppressed` `IN-REVIEW` developer — `NIRGraphEmitter_Dataflow.cpp::EmitParameterMapGet` and `EmitParameterMapSet` now skip the dynamic "Add" sentinel pin via `UNiagaraNodeWithDynamicPins::IsAddPin`. The Add pin is a UI affordance for adding new typed parameters, carries no `FNiagaraTypeDefinition`, and was emitting as `get $Add : Unknown` on every MapGet/MapSet node despite the engine itself excluding it from compilation (`IsValidPinToCompile`). Added regression tests `FNiagaraNirGraphParameterMapGetAddPinSuppressedTest` and `FNiagaraNirGraphParameterMapSetAddPinSuppressedTest` (`Tests/Niagara/TestNIRGraphDataflow.cpp`). `nir.txt` aspect version bumped in `AssetDumpCache.cpp` (shared with T2's centralized sprint bump).
- `#5-verify-fix` `DONE` tester — Verified: fresh `asset.dump` on `/Game/Effects/NiagaraModules/NM_ProjectToPlane` (the ticket's named repro) produced `nir.txt` with 0 matches for `: Unknown` and 0 matches for `$Add` (previously 12 instances of `get $Add : Unknown`). Re-dumped a second NDI-heavy asset `/Game/Effects/Particles/Item/NS_GunPad_Loading` to sanity-check generalization: also 0 matches for `: Unknown`. Test: `mcp__editor-automation__call` method=`asset.dump`, then grep produced `nir.txt`.
