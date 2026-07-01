---
id: B-crir-curves-not-emitted
title: "Control Rig IR omits Curves entirely (100-250 per rig)"
status: DONE
severity: High
category: bug
tags: [crir, control-rig, decompiler]
---

# CRIR omits Curves entirely

`crir.txt` for control rigs preserves bone location/rotation but completely drops Curve elements. Curve elements drive IK blending, hand/leg/spine deformation, and other rig logic that's load-bearing for understanding rig behavior.

The output shows `# TODO unsupported element kind: Curve` for every curve — 16 of 18 CRIR files in the dump tree show this pattern with 100-250 curves per rig.

## Sample

- `App/ThirdPerson/Characters/Rigs/CR_Mannequin_BasicFootIK/crir.txt` — 248+ curve TODOs
- Other CR files in `Game\USCS\` similarly affected

## Fix sketch

Control Rig hierarchy curves are flat `FRigCurveElement` float values, not keyed animation `FRichCurve` tracks. Add `curve` to CRIR hierarchy element coverage and emit each curve through the existing flat hierarchy element path:

```
curve IKBlend value=0.75
```

The decompiler should classify `ERigElementType::Curve`, read `URigHierarchy::GetCurveValue`, and emit `value=`. The parser/compiler should accept `curve`, default a missing `value=` to `0.0`, and recreate it with `URigHierarchyController::AddCurve(Name, Value, false, false)`. Do not add `default`, `evaluationMode`, or `keys`; those fields belong to keyed animation curves, not Control Rig hierarchy curves.

## History
- `#1-curves-todo-unsupported` `OPEN` reporter — 16/18 CRIR files affected; control-rig analysis is incomplete without curve data. High severity for rig inspection workflows.
- `#2-curve-hierarchy-roundtrip` `IN-REVIEW` developer — Added CRIR curve hierarchy opcode, grammar, parser, decompiler, compiler, docs, and a curve round-trip regression test so hierarchy curves emit as flat value elements instead of TODOs.
- `#3-verify-curves-emitted` `DONE` tester — Verified: fresh `asset.dump` on `/App/ThirdPerson/Characters/Rigs/CR_Mannequin_BasicFootIK` produced a crir.txt with 128 `curve <Name> value=0` lines (matching the 128 curve elements that were previously `TODO unsupported element kind: Curve`) and zero remaining TODO lines.
