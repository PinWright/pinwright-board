---
id: B-anim-curves-read-runtime-copy
title: "animation.authoring.list_curves and animation.describe_sequence read curves from the deprecated GetCurveData() runtime copy, which can be empty or stale for a freshly authored clip"
status: DONE
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
- `#2-shared-data-model-curve-read` `IN-REVIEW` developer — Every plugin `GetCurveData()` reader now goes through one shared helper pair, `AnimSequenceDumpBuilder::GetAuthoredFloatCurves` / `GetAuthoredTransformCurves` (`Source/PinWright/Private/Handlers/Asset/AnimSequenceDumpBuilder.h/.cpp`): the data model's `GetFloatCurves()` / `GetTransformCurves()` when `IsDataModelValid()`, the runtime copy otherwise. Readers moved onto it: `BuildCurvesArrayJson` (so `animation.authoring.list_curves`, the `curves` arrays on the other `animation.authoring.*` responses, `animation.describe_sequence`, `asset.dump` `anim_sequence.json`) and `animation.check_morph_curves` (`Handlers/Animation/MorphCurveAuditHandler.cpp`, its inline read replaced by the helper). A source sweep found no other `GetCurveData()` reader; `Utils/PropertyExport.cpp`'s `FRawCurveTracks` summary is a reflection export of the `RawCurveData` property itself and stays as is. Reproduced deterministically rather than live: inside an open controller bracket the engine does not copy model curves into `RawCurveData` (`AnimSequenceBase.cpp` `OnModelModified`, `NotifyCollector.IsNotWithinBracket()`), and on load the copy has redundant keys stripped, so keyCount differed too. `anim_sequence.json` aspect version 3 -> 4 (`AssetDumpCache.cpp`, pinned by `PinWright.AssetDumpCache.AnimSequenceAspectVersion`). Test: `PinWright.Assets.AnimSequence.DumpBuilder.CurvesReadDataModel` (`Tests/Assets/TestAnimSequenceDumpBuilder.cpp`) adds a 2-key float curve and a transform curve inside an open bracket, asserts the precondition that the runtime copy is empty and the model holds both, and asserts `curves[]` lists both with the model's keyCount (old read: empty). Wiki: `animation.authoring.md` curves paragraph. CHANGELOG entry.
- `#3-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.Assets.AnimSequence.DumpBuilder.CurvesReadDataModel`, `PinWright.AssetDumpCache.AnimSequenceAspectVersion`, `PinWright.animation.check_morph_curves.FlagAndNameDecideDrives`. Acceptance (read curves from the data model when `IsDataModelValid()`, else the runtime copy) met. The test adds float and transform curves inside an open controller bracket and asserts the precondition that the runtime copy is empty while the model holds both. It then asserts `curves[]` lists both with the model's keyCount, so the read behind `list_curves`, `describe_sequence` and `anim_sequence.json` is covered by the shared builder. The `anim_sequence.json` aspect bump 3 -> 4 is pinned.
