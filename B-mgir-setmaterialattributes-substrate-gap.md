---
id: B-mgir-setmaterialattributes-substrate-gap
title: "MGIR SetMaterialAttributes nodes only emit BaseColor + Normal (Substrate gaps)"
status: DONE
severity: Low
category: bug
tags: [mgir, material, decompiler, substrate]
---

# MGIR SetMaterialAttributes only emits BaseColor + Normal

SetMaterialAttributes nodes in MGIR emit only 2-3 attributes (BaseColor, Normal) with an `AttributeSetTypes` GUID array. Substrate material layer data, height, depth offset, emissive, custom attributes, and other pins are dropped.

Substrate shading models (added UE5.1+) and custom material attributes are underrepresented.

## Sample

`App/Brushify/Materials/Landscape/Materials/Grass/MF_Grass/mgir.txt` — line 10 shows only BaseColor + Normal emitted; landscape-grass material has more attribute pins authored.

## Fix sketch

`MGIRDecompiler.cpp` should special-case `UMaterialExpressionSetMaterialAttributes`: resolve `AttributeSetTypes` through `FMaterialAttributeDefinitionMap`, emit a stable named `Attributes: [...]` list for every authored attribute pin, and emit connected attribute inputs by display name. Pair this with `MGIRCompiler.cpp` support that consumes `Attributes` and recreates the dynamic pins before pending wires are applied. Do not emit fake source expressions for unconnected authored pins; preserve them through the named attribute list.

## History
- `#1-setmatattr-partial` `OPEN` reporter — affects all materials using SetMaterialAttributes with non-default attribute lists.
- `#2-fix-sketch-corrected` `OPEN` developer — Bug is real, but the decompiler-only GUID-decoding sketch is incomplete; MGIR also needs compile-side dynamic pin recreation and unconnected pins should be preserved as named attribute metadata.
- `#3-named-attributes-roundtrip` `IN-REVIEW` developer — Added SetMaterialAttributes MGIR named attribute emission, compile-side dynamic pin recreation, and a regression test covering authored connected and unconnected attribute pins.
- `#4-verify-roundtrip` `DONE` tester — Verified: fresh asset.dump on `/App/Brushify/Materials/Landscape/Materials/Grass/MF_Grass` now emits `Attributes: [BaseColor,Normal]` named list (old GUID-array `AttributeSetTypes` gone). Round-trip on temp `/Game/App/UI/Test/MF_McpVerifyTemp_setmatattr` with 4 attributes (BaseColor, Metallic, Roughness, EmissiveColor) compiled (5 expressions created), redumped MGIR preserved the full named list and all 4 wires, temp asset deleted.
