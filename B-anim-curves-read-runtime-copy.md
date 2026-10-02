---
id: B-anim-curves-read-runtime-copy
title: "animation.authoring.list_curves and animation.describe_sequence read curves from the deprecated GetCurveData() runtime copy, which can be empty or stale for a freshly authored clip"
status: OPEN
severity: Medium
category: bug
tags: [animation, curves, anim-sequence, data-model, stale-data, describe-sequence, list-curves]
encounters: 1
lastSeen: 2026-10-02T00:21:17Z
---

# Curve listings read the runtime curve copy, not the data model

`AnimSequenceDumpBuilder::BuildCurvesArrayJson`
(`Source/PinWright/Private/Handlers/Asset/AnimSequenceDumpBuilder.cpp:165` and `:173`) builds
`curves[]` from `UAnimSequenceBase::GetCurveData().FloatCurves` / `.TransformCurves`. That is the
deprecated runtime `RawCurveData` copy, which the engine syncs from the animation data model only on
some notifications. The data model (`GetDataModel()->GetFloatCurves()`) is the editor authority.

Every caller of the builder inherits the read:

- `animation.authoring.list_curves` (`AnimationAuthoringHandler_Sequence.cpp:885`), plus the
  `curves` arrays other `animation.authoring.*` responses attach through the same builder
  (`:1152`, `:1578`, `:1628`, `:1655`);
- `animation.describe_sequence` (`AnimSequenceDescribeHandler.cpp`, via
  `BuildAnimSequenceJson`);
- `asset.dump`'s `anim_sequence.json` (same `BuildAnimSequenceJson`).

`animation.check_morph_curves` had the same read and returned an empty curve set for a sequence
whose data model held four float curves, so it reported a false green. It was fixed to read the
data model (see History). These verbs were not checked live, so whether they show the same
symptom is unconfirmed. If they do, a caller authoring curves and then listing them gets a silent
wrong answer: curves missing, or a stale key count.

**Fix:** read the data model's curves (`GetDataModel()->GetCurveData()`, float and transform)
when `IsDataModelValid()`, falling back to `GetCurveData()` otherwise, as
`Handlers/Animation/MorphCurveAuditHandler.cpp` now does for float curves.

## History
- `#1-same-read-as-morph-audit` `OPEN` reporter — Found while fixing `F-animation-check-morph-curves` (`#4-fixround2-read-data-model`). In suite run w23-full2 that verb's `GetCurveData()` read came back empty for a sequence whose data model held four float curves. The fix in `Source/PinWright/Private/Handlers/Animation/MorphCurveAuditHandler.cpp` (plugin commit `ce451931`) reads `GetDataModel()->GetFloatCurves()` when `IsDataModelValid()`. `AnimSequenceDumpBuilder::BuildCurvesArrayJson`, behind `animation.authoring.list_curves` and `animation.describe_sequence`, still reads `GetCurveData()`. Neither verb was checked live against a freshly authored clip, so this is a code-read finding: same read, same possible staleness. Severity Medium (silent wrong data if it reproduces, unconfirmed).
