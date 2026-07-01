---
id: B-asset-dump-rawcurvedata-text-blob
title: "asset.dump AnimSequence RawCurveData rendered as multi-thousand-keyframe single-line ExportText blob"
status: DONE
severity: Medium
category: bug
tags: [asset-dump, animation, properties-blob]
---

# asset.dump AnimSequence RawCurveData rendered as multi-thousand-keyframe single-line ExportText blob

`RawCurveData` (FRawCurveTracks) is exported via UE's ExportText, producing a multi-thousand-keyframe string on a single line. Property output for an AnimSequence is dominated by this blob (megabyte-scale per asset).

Practically unreadable for any LLM workflow, cheap to overflow context.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonIdle/properties.json` (line 226).
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/TheSlowMoKit/Demo/Mannequin/Animations/ThirdPersonIdle/properties.json`.
3. Observe: `RawCurveData` is a single-line escaped struct string with thousands of keyframes; dominates the file.

**Fix:** Serialize FRawCurveTracks as a compact structured summary in properties.json when the generic property exporter sees RawCurveData: include curve counts, total key counts, and per-curve key counts plus bounded first/last key metadata. Do not dump one JSON object per keyframe by default; that only moves the context-heavy data to a sidecar. If full keyframe export is needed later, file a separate opt-in feature with size limits.

## History
- `#1-initial-repro` `OPEN` reporter — AnimSequence `RawCurveData` dumps as a single-line escaped ExportText blob containing thousands of keyframes; megabyte-scale per asset. Sample paths: `Game/SoStylized/Demo/Pawn/Mannequin/Animations/ThirdPersonIdle/properties.json` (line 226), `Game/TheSlowMoKit/Demo/Mannequin/Animations/ThirdPersonIdle/properties.json`.
- `#2-compact-rawcurvedata-summary` `IN-REVIEW` developer — Replaced the FRawCurveTracks ExportText fallback in Utils/PropertyUtils.cpp with a compact structured summary for RawCurveData, avoiding single-line keyframe blobs in asset.dump properties.json; added FPropertyUtilsRawCurveDataSummaryTest to assert curve/key counts and absence of the ExportText Keys blob.
- `#3-verify-fix` `DONE` tester — Verified: re-ran asset.dump on both repro assets (SoStylized + TheSlowMoKit ThirdPersonIdle); RawCurveData now renders as structured JSON with floatCurveCount/floatCurves[]/totalKeyCount/transformCurveCount and per-curve keyCount+firstKey+lastKey; zero `Keys=(` ExportText substrings in either properties.json.
