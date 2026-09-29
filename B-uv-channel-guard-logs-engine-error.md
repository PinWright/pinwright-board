---
id: B-uv-channel-guard-logs-engine-error
title: "Out-of-range uvChannel reaches GeometryScript SetNumUVSets, which logs an engine Error before the verb refuses"
status: IN-REVIEW
severity: Low
category: bug
tags: [geometry, uv, log-noise, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T10:23:06Z
---

# `uvChannel >= 8` logs `SetNumUVSets: Maximum of 8 UV Sets are supported`

`GeometryUtils::EnsureMeshHasUVChannel` (`Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryUtils.cpp`) detected an
out-of-range channel by calling `UGeometryScriptLibrary_MeshUVFunctions::SetNumUVSets(Mesh, UVChannel + 1, nullptr)` and
checking whether the layer appeared. The engine refuses `> 8` sets through `AppendError`, which logs
`LogGeometry: Error: SetNumUVSets: Maximum of 8 UV Sets are supported` whether or not a debug sink is passed
(`MeshUVFunctions.cpp:39-42`, `GeometryScriptTypes.cpp:26-33`). The verbs then refused correctly with INVALID_ARGUMENT, so the
only effect was an engine `Error:` line beside the structured refusal on every `project_uv` / `unwrap_uv` / `auto_uv` /
`pack_uv_islands` / `layout_uv` / patch-builder / `.pwmodel` call with channel 8+.

Observed in `Geometry.Ops.Modeling.UVOpsRejectAnOutOfRangeChannel`, `Geometry.Ops.WidenedOptions.LayoutAndPatchBuilderRejectAnOutOfRangeChannel`,
`geometry.uv_generation.CreatesUVLayerOnUVLessMesh` (2 lines each), `Saved/Logs/pw_gapwave_full_offscreen2.log:31471,31819,31976`.

**Fix:** refuse `UVChannel >= 8` in `EnsureMeshHasUVChannel` before any engine call.

## History
- `#1-setnumuvsets-logs-error` `OPEN` reporter — Found while triaging tests newly exposed by the per-test `bSuppressLogErrors` reset: the channel-range guard probed the engine with the invalid count and the engine logged an Error.
- `#2-range-check-before-engine` `IN-REVIEW` developer — `EnsureMeshHasUVChannel` now returns false for `UVChannel < 0 || UVChannel >= 8` without calling `SetNumUVSets`; header comment updated. All callers already turn false into their INVALID_ARGUMENT "out of range" refusal, so responses are unchanged. Regression check: the three tests above fail on the engine error if the check is reverted. Compile-checked (`-SingleFile`) only; needs a suite run.
- `#3-exact-expected-error-counts` `IN-REVIEW` developer — Same triage, adjacent sites (docs/rpc-design.md section 12 now forbids `Occurrences=0`). Replaced all 12 `Occurrences=0` declarations with exact counts: `TestGeometryBooleanFailureDetection.cpp` EnclosedSubtract 2, DisjointIntersection 2, Trim 1, FailedBooleanReachesTheCaller 2, FailedBooleanDoesNotDestroyTheToolActor 2; `TestGeometryEngineFailureDetection.cpp` DuplicateFace 2, NonManifoldEdge 2, AllowPartial 2 (two imports, camel and snake flag), RefusedAppendEmpty 2, RefusedAppendRestores 2, Tangents 1, XAtlasNonCompact 1. Every pattern fires. Why 2 instead of 1: engine Error/Warning lines route through GWarn, where `FAutomationTestMessageFilter` matches the BARE message against expectations (and counts it) regardless of `bSuppressLogWarnings`, so a PinWright `LogMcpGeometryHandlersNew` Warning quoting the engine sentence counts too. Counts derived from the code paths and confirmed against `Saved/Logs/pw_gapwave_full_offscreen2.log`, where each matched line appears downgraded to `Verbose` (30107-31488). `TestGeometryWidenedOpOptions.cpp` BooleanAllowEmptyResultReachesTheEngine corrected to the bare `BooleanUnion: Boolean operation failed`, count 2 (a `LogGeometry: ` prefix never matches at the filter). Compile-checked (`-SingleFile`) only; needs a suite run.
