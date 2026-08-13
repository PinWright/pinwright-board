---
id: B-mgir-linearcolor-array
title: "MGIR rejects its own LinearColor array output"
status: IN-REVIEW
severity: Medium
category: bug
tags: [material, mgir, round-trip, property-import]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# MGIR rejects its own LinearColor array output

`material.decompile_mgir` serializes `FLinearColor` properties such as
`UMaterialExpressionVectorParameter.DefaultValue` as a four-number array, but
`material.compile_mgir` only accepts array-shaped reflected structs for Vector
and Rotator. Recompiling the decompiler's valid output therefore rejects a
vector-parameter default instead of preserving it.

**Workaround:** Rewrite the array into an object-shaped `FLinearColor` value or
set the value through a separate imperative property mutation.

**Fix:** Extend the shared reflected-property importer to accept three- or
four-number arrays for `FLinearColor`, defaulting alpha to 1 when omitted, and
cover a decompile/compile round trip with a focused automation test.

## History
- `#1-linearcolor-array-repro` `OPEN` reporter — Reproduced while authoring the DroneFootball arena glass material: MGIR decompiled a VectorParameter `DefaultValue` to `[R, G, B, A]`, but compiling that text rejected the property. Source inspection confirms the shared array importer handles Vector and Rotator but not LinearColor.
- `#2-accept-linearcolor-arrays` `IN-REVIEW` developer — Extended `ApplyJsonValueToProperty` to accept three- or four-number arrays for `FLinearColor`, added a true vector-parameter decompile/compile round-trip automation test, and documented the accepted MGIR shape. UE 5.8 standalone `BuildPlugin` compiled all 1043 actions successfully with exit code 0.
