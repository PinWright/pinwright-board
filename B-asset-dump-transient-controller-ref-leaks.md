---
id: B-asset-dump-transient-controller-ref-leaks
title: "asset.dump AnimSequence properties.json leaks /Engine/Transient.AnimSequencerController_<NNN> refs (defeats diff-baseline stability)"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, animation, transient, diff-stability]
---

# asset.dump AnimSequence properties.json leaks /Engine/Transient.AnimSequencerController_<NNN> refs (defeats diff-baseline stability)

AnimSequence `Controller` UPROPERTY is dumped as `/Engine/Transient.AnimSequencerController_143` — a session-scoped transient editor object whose numeric suffix varies per editor instance.

Every dump pass produces a different value, defeating diff-baseline stability.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonIdle/properties.json` (line ~136).
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/TheSlowMoKit/Demo/Mannequin/Animations/ThirdPersonIdle/properties.json`.
3. Observe: `Controller` field references `/Engine/Transient.AnimSequencerController_143` (suffix varies per session).

**Fix:** Suppress object/interface properties from asset-dump properties.json when the property is transient/duplicate-transient and the resolved UObject lives in the transient package. Do not redact numeric suffixes; /Engine/Transient references are editor session state, not stable asset data.

## History
- `#1-initial-repro` `OPEN` reporter — AnimSequence `Controller` dumps `/Engine/Transient.AnimSequencerController_<NNN>` with a session-scoped numeric suffix; every dump pass produces a different value. Sample paths: `Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonIdle/properties.json` (line ~136), `Game/TheSlowMoKit/Demo/Mannequin/Animations/ThirdPersonIdle/properties.json`. 13/13 AnimSequence dumps in slice.
- `#2-suppress-transient-controller` `IN-REVIEW` developer — Suppressed transient-package object/interface property refs from asset.dump properties output so AnimSequence Controller no longer emits session-scoped /Engine/Transient paths. Added regression coverage in TestAssetDumpAnimSequenceProperties.cpp.
- `#3-verify-fix` `DONE` tester — Verified: ran asset.dump on both repro assets (/Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonIdle and /Game/TheSlowMoKit/Demo/Mannequin/Animations/ThirdPersonIdle); freshly-dumped properties.json contains no `Controller` field and zero `/Engine/Transient.AnimSequencerController_*` references (grep matches only the unrelated `DuplicateTransient`/`Transient` flag tokens on numeric properties).
