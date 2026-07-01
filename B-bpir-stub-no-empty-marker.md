---
id: B-bpir-stub-no-empty-marker
title: "BPIR header-only stubs are indistinguishable from decompile failures"
status: DONE
severity: Medium
category: bug
tags: [bpir, asset-dump, observability, emitter]
---

# BPIR header-only stubs are indistinguishable from decompile failures

`BuildBpirText` in `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Utils/AssetDumpBuilder.cpp:107-123` always emits a `# ==== Graph: <Name> (<Kind>) ====` header even when `FBpirDecompiler::DecompileGraph` returns an empty `BpirText` with zero warnings. The result is a `bpir.txt` containing only the header (43 bytes), with no marker distinguishing between:

- **(a)** Data-only Blueprint where the auto-created EventGraph has zero placed nodes — the common case for `UCameraShakeBase` subclasses, data-only `UActorComponent` subclasses, etc. This is correct behavior.
- **(b)** A latent decompile failure that swallowed an error and returned empty.

A consumer (LLM agent, audit tool) cannot tell these apart without inspecting the source asset. An earlier coverage audit miscounted ~500 such stub files as "missing/empty BPIR" when they were actually case (a) — correct behavior — but indistinguishable from case (b) without source inspection.

This mirrors the MGIR analog already fixed by `B-asset-dump-mgir-empty-graph-ambiguous` (DONE), which added `# no expression graph` markers to empty material graph dumps. BPIR needs the same disambiguation surface.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/Drone/DroneCameraShake/bpir.txt` — 43 bytes, header only.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/LevelBlueprints/B_DroneMusicManager/bpir.txt` — same shape.
3. Compare to `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/App/App/LevelBlueprints/B_DroneGameMode/bpir.txt`, which has the same header line followed by real graph bodies.

Both repro files contain only:
```
# ==== Graph: EventGraph (ubergraph) ====
```
with no body, indistinguishable from a silently-failed decompile.

**Fix:** In `BuildBpirText`'s per-graph append loop, when `Result.BpirText.IsEmpty() && Result.Warnings.Num() == 0`, probe `Graph->Nodes.Num()` and emit one of:
- `# (empty: no event/function bodies)` — when `Graph->Nodes.Num() == 0`
- `# (no decompiled bodies)` — when `Graph->Nodes.Num() > 0` (entry-only or hidden nodes)

When `Result.Warnings.Num() > 0` or the decompiler signals failure, emit `# BPIR_FAILED: <first-warning>` so grep can detect it. Optionally extend `FBpirDecompileResult` with a `bool bFailed` flag so the emitter can disambiguate without inspecting warning text.

## History
- `#1-initial-repro` `OPEN` reporter — `BuildBpirText` (`AssetDumpBuilder.cpp:107-123`) unconditionally emits the `# ==== Graph: <Name> (<Kind>) ====` header before appending `Result.BpirText`, so an empty decompile (no warnings, no body) produces a 43-byte header-only `bpir.txt`. Sample paths: `App/App/Drone/DroneCameraShake/bpir.txt` and `App/App/LevelBlueprints/B_DroneMusicManager/bpir.txt` — both 43 bytes containing only `# ==== Graph: EventGraph (ubergraph) ====`. Compare `App/App/LevelBlueprints/B_DroneGameMode/bpir.txt` for the same header followed by real bodies. Case (a) data-only BPs with auto-created empty EventGraphs and case (b) silent decompile failures are textually identical. Earlier coverage audit miscounted ~500 such stubs as "missing/empty BPIR" when they were actually correct. Propose an empty/failure marker inside the header block, mirroring `B-asset-dump-mgir-empty-graph-ambiguous`'s `# no expression graph` marker for MGIR.
- `#2-add-empty-marker-bpir` `IN-REVIEW` developer — Updated `AssetDumpBuilder.cpp::BuildBpirText`'s `AppendGraph` lambda so empty decompile output emits a disambiguation marker after the graph header: `# BPIR_FAILED: <warning>` when `Result.Warnings.Num() > 0`, `# (empty: no event/function bodies)` when `Graph->Nodes.Num() == 0`, `# (no decompiled bodies)` otherwise. Mirrors `B-asset-dump-mgir-empty-graph-ambiguous` MGIR fix. Regression test `TestBpirEmptyGraphMarker.cpp::FBpirBuildBpirTextEmptyGraph_EmitsEmptyMarker` builds a transient BP with an empty EventGraph, calls production `BuildBpirText`, asserts the header AND one of the two empty markers appears. Counterfactual: reverting the marker emission collapses output back to header-only and the marker-substring assertion fails.
- `#3-verify-empty-marker` `DONE` tester — Verified: re-dumped both repro assets via `asset.dump` on `/App/App/Drone/DroneCameraShake` and `/App/App/LevelBlueprints/B_DroneMusicManager`; both `bpir.txt` files now contain the header followed by `# (empty: no event/function bodies)` instead of header-only output.
