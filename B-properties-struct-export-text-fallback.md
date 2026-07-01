---
id: B-properties-struct-export-text-fallback
title: "Unrecognized struct types fall back to ExportText string, losing structure"
status: DONE
severity: Medium
category: bug
tags: [properties, structs, serialization]
---

# Unrecognized struct types fall back to ExportText string

Distinct from the array-of-struct case in [B-asset-dump-struct-array-as-export-text-strings](B-asset-dump-struct-array-as-export-text-strings.md) — this is about *single-valued* struct properties.

When a struct property's type isn't explicitly handled by the JSON struct-converter, the value is emitted as UE's parenthesized export-text format inside a string field rather than being decomposed into a structured JSON object. Examples seen in the wild:

- `FActorComponentTickFunction` → `"value": "(...)"` literal
- `FBodyInstance` → giant unparsed paren-string
- `FBox` → `"value": "()"` (empty paren — apparently a degenerate case)
- `FMaterialAttributesInput` → `"FrontMaterial": "()"`, `"MaterialAttributes": "()"`
- `FNiagaraUserRedirectionParameterStore` → 100+ hex bytes for `ParameterData` packed inline

Sample paths:
- `Game/Blueprints/Pawn/ThirdPersonCharacter/properties.json` (BodyInstance)
- `Game/Blueprints/M_MoviePipelineRenderPreview/properties.json` (FrontMaterial, MaterialAttributes)
- `Game/Effects/Particles/Item/NS_GunPad_Loading/properties.json` (ExposedParameters, SystemCompiledData)

## Fix

`PropertyUtils.cpp`'s struct-to-JSON converter has an ExportText fallback for unrecognized struct types. Add reflection-based field iteration so unknown structs are walked as JSON objects (one entry per struct property) rather than emitted as opaque export-text strings. Document which struct types are still intentionally serialized as strings (e.g., `FGuid`, where a string form is the canonical representation).

## History
- `#2-single-struct-reflection` `IN-REVIEW` implementer — generic single `FStructProperty` fallback now uses reflected `StructToJsonObject`; existing explicit compact/special cases such as vector-shaped properties remain compact. Added utility coverage for `FActorComponentTickFunction`-style reflected struct output.
- `#1-struct-fallback-leak` `OPEN` reporter — found in actor/BP/data-asset dumps across the tree. Affects FActorComponentTickFunction, FBodyInstance, FMaterialAttributesInput, FNiagaraUserRedirectionParameterStore among others. Distinct from B-asset-dump-struct-array-as-export-text-strings (arrays) — this is single-valued structs.
- `#3-verify-fix` `DONE` tester — Verified: ran `asset.dump` on `/Game/Blueprints/Pawn/ThirdPersonCharacter`; in resulting properties.json, `BodyInstance` and `PostPhysicsTickFunction` now appear as JSON objects with named fields (e.g. `AngularDamping`, `bAllowTickOnDedicatedServer`) instead of opaque `"(...)"` strings; zero `": "("` opaque-paren occurrences remain in the dump.
