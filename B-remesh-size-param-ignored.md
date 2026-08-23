---
id: B-remesh-size-param-ignored
title: "geometry.quadrangulate + geometry.remesh_voxel silently ignore their size-control parameter (targetQuadSize / voxelSize) — both hardcode a TrisBefore/2 uniform remesh, so any requested density is a no-op and the call reports success (remesh_voxel even echoes the ignored voxelSize back)"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, quadrangulate, remesh_voxel, remesh-size-param-ignored, silent-no-op, ignored-param, placeholder-remesh, hardcoded]
encounters: 1
lastSeen: 2026-07-13T09:45:51.5624147+03:00
---

# quadrangulate/remesh_voxel disregard their density parameter and hardcode TrisBefore/2

Two retopology verbs in `AdvancedMeshOpsHandler.cpp` are placeholder
implementations that **never read their documented size-control parameter** and
instead run a fixed `TargetTriangleCount = max(100, TrisBefore/2)` uniform remesh.
Passing different density values produces byte-identical output, and the call
still returns success — a silent no-op on the one knob each verb exists to
provide.

## Affected methods (same root cause)

- `geometry.quadrangulate` — documents `targetQuadSize`, `preserveFeatures`,
  `featureAngleThreshold`. The handler reads **none** of them; the density is a
  hardcoded `TrisBefore/2`. (Reproduced live — see below.)
- `geometry.remesh_voxel` — documents `voxelSize`, `surfaceDistance`, `fillHoles`.
  The handler reads `voxelSize` but **never uses it**, and never reads
  `surfaceDistance` at all; only `fillHoles` is honored, over the same hardcoded
  `TrisBefore/2` remesh. Worse: it **echoes the ignored `voxelSize` back in the
  success response**, actively implying it was applied. (Reproduced live — see
  below.)

Enumerated via a full source sweep for the shared `FMath::Max(100, TrisBefore / 2)`
pattern (`AdvancedMeshOpsHandler.cpp:1188` and `:1238`); those are the only two
occurrences. `geometry.remesh_uniform` (a different handler, `MeshOpsHandler.cpp`)
genuinely honors its `targetTriangleCount` and is NOT part of this family — its
issue is a separate response-echo ergonomic (see `E-remesh-uniform-echoes-target-not-achieved`).

## Guilty source lines (verbatim, ground truth)

`geometry.quadrangulate` — `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`:

```cpp
1172:    FString ActorName = Ctx.GetString(TEXT("actorName"));   // the ONLY param read
...
1187:    FGeometryScriptUniformRemeshOptions UniformOptions;
1188:    int32 TargetTris = FMath::Max(100, TrisBefore / 2);      // hardcoded; targetQuadSize never referenced
1189:    UniformOptions.TargetType = EGeometryScriptUniformRemeshTargetType::TriangleCount;
1190:    UniformOptions.TargetTriangleCount = TargetTris;
1191:
1192:    UGeometryScriptLibrary_RemeshingFunctions::ApplyUniformRemesh(Mesh, RemeshOptions, UniformOptions, nullptr);
...
1202:    Result->SetStringField(TEXT("note"), TEXT("Partial quadrangulation applied - full quad remesh requires external library"));
```

The `note` discloses that it is not a true quad remesh, but it does NOT disclose
that `targetQuadSize` (the density knob) has zero effect. `preserveFeatures` and
`featureAngleThreshold` are likewise never read.

`geometry.remesh_voxel` — same file:

```cpp
1222:    double VoxelSize = Ctx.GetNumber(TEXT("voxelSize"), 10.0);   // read...
1223:    bool bFillHoles = Ctx.GetBool(TEXT("fillHoles"), true);
...
1237:    FGeometryScriptUniformRemeshOptions UniformOptions;
1238:    int32 TargetTris = FMath::Max(100, TrisBefore / 2);          // ...but VoxelSize used NOWHERE below
1239:    UniformOptions.TargetType = EGeometryScriptUniformRemeshTargetType::TriangleCount;
1240:    UniformOptions.TargetTriangleCount = TargetTris;
1241:
1242:    UGeometryScriptLibrary_RemeshingFunctions::ApplyUniformRemesh(Mesh, RemeshOptions, UniformOptions, nullptr);
...
1260:    Result->SetNumberField(TEXT("voxelSize"), VoxelSize);        // echoes the ignored input back
```

## Verbatim replay (this host, live HEAD)

Baseline torus: `geometry.create_torus {name, majorRadius:100, minorRadius:30}` →
`geometry.get_mesh_info` → `{vertexCount:128, triangleCount:256}`.

`geometry.quadrangulate` — targetQuadSize is a no-op (identical output for 40 vs 500 on fresh identical tori):

- `{actorName:OracleQuadTorusA, targetQuadSize:40}` → `{trianglesBefore:256, trianglesAfter:262, note:"Partial quadrangulation applied - full quad remesh requires external library"}`
- `{actorName:OracleQuadTorusB, targetQuadSize:500}` → `{trianglesBefore:256, trianglesAfter:262, note:"Partial quadrangulation applied - full quad remesh requires external library"}`

A 12.5x change in targetQuadSize yields the exact same 262-triangle result. The
attempt's own control (fresh torus, absurd targetQuadSize=200) likewise matched
its targetQuadSize=40 result byte-for-byte.

`geometry.remesh_voxel` — voxelSize is a no-op (identical output for 2 vs 200 on fresh identical tori), and the ignored value is echoed:

- `{actorName:OracleVoxTorusC, voxelSize:2}` → `{voxelSize:2, trianglesBefore:256, trianglesAfter:262}`
- `{actorName:OracleVoxTorusD, voxelSize:200}` → `{voxelSize:200, trianglesBefore:256, trianglesAfter:262}`

A 100x change in voxelSize yields the same 262-triangle result; the response
faithfully echoes `voxelSize:2` / `voxelSize:200` as if it were applied.

## Impact

The attempted task (retopologize a torus prop, coarse pass then finer pass,
verifying the finer target is strictly denser) is impossible: no value of
`targetQuadSize` changes the output, so the "coarse must differ from baseline and
finer must be strictly denser than coarser" density round-trip can never be
satisfied through these verbs. A caller trusts a lie — the tool reports success and
(for remesh_voxel) echoes the requested size, while silently discarding it. There
is no in-verb workaround; density control requires a different verb entirely
(`geometry.remesh_uniform` with `targetTriangleCount`, or `geometry.simplify_mesh`).

## What it should do

Wire each documented size parameter into the remesh. Uniform remesh supports a
target-edge-length mode (`EGeometryScriptUniformRemeshTargetType::TargetEdgeLength`)
— map `targetQuadSize`/`voxelSize` onto it (or, for remesh_voxel, use the real
voxel path `ApplyMeshSolidify`/voxel remesh) so a smaller size yields a denser mesh
and a larger size a coarser one. At minimum, if a parameter genuinely cannot be
honored on the current engine, do not accept-and-silently-ignore it: reject with a
clear error or document the no-op in the `note`/response (and stop echoing an
unused `voxelSize` as though it were applied).

## Dedup

Ripgrep of the board for `quadrangulate`/`targetQuadSize`/`remesh_voxel`/`voxelSize`
found no existing ticket. `E-remesh-uniform-echoes-target-not-achieved` (IN-REVIEW)
is a DIFFERENT handler (`remesh_uniform`, `MeshOpsHandler.cpp`) with a different
root cause (it honors its target but reports the requested rather than achieved
count) — verified by opening it; not a dupe.

severity rationale: impact=silent false-success / silent hardcoded no-op of the
sole density-control parameter on a normal retopo path — the caller trusts a lie,
and remesh_voxel actively echoes the ignored value back → High × reach=retopology
is a mainline mesh-prep path (two affected verbs), not every-session and not a rare
corner → no bump → High

## History
- `#1-initial-repro` `OPEN` reporter — SEED `geometry.quadrangulate` (SEED mode). Attempt (torus retopo prop-prep, coarse then fine density) reported blocked_by_tool: targetQuadSize is a no-op. Replay-confirmed live at HEAD AND in source. quadrangulate (`AdvancedMeshOpsHandler.cpp:1172,1188`) reads only actorName and hardcodes `TargetTris=max(100,TrisBefore/2)`; targetQuadSize=40 and targetQuadSize=500 on fresh identical 256-tri tori both produced trianglesAfter=262. Family probe found sibling `geometry.remesh_voxel` (`:1222,1238`) shares the exact defect: reads voxelSize but never uses it (surfaceDistance never read), hardcodes the same TrisBefore/2 remesh, and echoes the ignored voxelSize in the response — voxelSize=2 and voxelSize=200 on fresh identical tori both produced trianglesAfter=262. Filed as a family ticket (tag `remesh-size-param-ignored`) naming both verbs. Culprit = geometry.quadrangulate (the seed). Severity High (silent false-success on the density knob, caller trusts a lie, no in-verb workaround).
- `#2-additional-docs-floor-wiki-overlay` `OPEN` struggle-auditor — DOCS-FLOOR angle (distinct from the code fix, same task/observation so no encounters bump). The pre-call discoverability gap: the wiki text the caller reads BEFORE calling carries no caveat that these verbs are partial placeholders whose size knob is ignored — the only disclosure is the runtime `note`, seen only AFTER the call. The geometry overlay is a single file `Plugins/PinWright/Docs/wiki-src/geometry.md`, which has NO `### geometry.quadrangulate` and NO `### geometry.remesh_voxel` section (the params the caller sees are auto-generated from RPC_PARAM defaults), so there is nowhere the limitation is stated up front. This forced the attempt into an extra 3-call control experiment (fresh torus + quadrangulate targetQuadSize=200 + get_mesh_info) purely to discover empirically what a one-line doc caveat would have stated. DOCS FLOOR (independent of, and useful until, the code fix): add `### geometry.quadrangulate` and `### geometry.remesh_voxel` overlay sections to `geometry.md` noting "partial-placeholder remesh — targetQuadSize/voxelSize (and preserveFeatures/featureAngleThreshold/surfaceDistance) are not yet honored; the density is a hardcoded TrisBefore/2. Use geometry.remesh_uniform for real density control." If the primary code fix wires the params in, drop the caveat instead. Evidence: CallAnalyzer inefficiency #2 (pattern wiki-nav) on this task; the runtime `note` already emits the limitation, the wiki does not.
- `#3-go-wire-size-onto-target-edge-length` `IN-REVIEW` developer — GO. Verified the defect at HEAD: quadrangulate reads only actorName, remesh_voxel reads voxelSize but applies it nowhere, and both hardcode `TargetTris=max(100,TrisBefore/2)` in a TriangleCount-mode uniform remesh (`AdvancedMeshOpsHandler.cpp` :1188/:1238); the adopted red test `PinWright.geometry.retopo.SizeParamHonored` reproduces it (dense==coarse). Decision: wire the PRIMARY size knob (targetQuadSize / voxelSize) onto the uniform remesh's `EGeometryScriptUniformRemeshTargetType::TargetEdgeLength` (enum value + `FGeometryScriptUniformRemeshOptions::TargetEdgeLength` field confirmed present in the UE 5.7 `MeshRemeshFunctions.h`), so a smaller size yields a denser mesh; this also makes remesh_voxel's existing voxelSize echo honest. Severity High unchanged. SCOPE NARROWED to the size knob only — the secondary declared-but-ignored params (quadrangulate's preserveFeatures/featureAngleThreshold, remesh_voxel's surfaceDistance) are split into new OPEN ticket `F-retopo-secondary-params-ignored` (they cannot be honored on the uniform-remesh path; drop-or-implement tracked there), not fixed here. Shipped: both handler bodies in `Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp` now drive the uniform remesh in `TargetEdgeLength` mode with `TargetEdgeLength=max(0.01f,(float)size)` from targetQuadSize / voxelSize; the `int32 TargetTris=max(100,TrisBefore/2)` hardcode is gone from both. Adopted red test `PinWright.geometry.retopo.SizeParamHonored` (`Tests/Geometry/TestGeometryRetopoSizeParamHonored.cpp`) asserts dense-request tris > coarse-request tris on fresh identical tori for BOTH verbs. Plugin built clean (Result: Succeeded, zero error lines) and the red test flipped Fail -> Result={Success} — red-green differential confirmed. Lead did not commit/push the plugin code; the commit phase lands one plugin commit.
- `#4-attempt-abandoned` `OPEN` supervisor — attempt-failed: fuzz3's run was stopped (user fleet stop, 2026-07-13) between the verified implement and the commit phase; the fix never reached the plugin origin and the uncommitted diff was swept by the fleet hard-sync. Reopened, lease released. The `#3` verified analysis and fix approach (TargetEdgeLength wiring, red test `PinWright.geometry.retopo.SizeParamHonored`) remain valid as a starting point.
- `#5-additional-current-removal-audit` `OPEN` reporter — Additional evidence: **Adversarial review A — CLOSE CANDIDATE.** Actuality: STALE/FIXED. Framing: the historical defect was real before the audit, but current HEAD no longer registers either `geometry.quadrangulate` or `geometry.remesh_voxel`: the live advanced handler lists only bridge/loft/sweep/extrude/duplicate/edge-split, while the retained remesh verb is `geometry.remesh_uniform`; Git history `e0d0fe2c` (2026-07-14) removed both methods as part of the dead/superseded RPC audit. High severity therefore does not apply to the current API unless these capabilities are intentionally restored. Proposed fix: INCOMPLETE, the prior TargetEdgeLength wiring is moot after removal and would only be a valid size mapping for a deliberately reintroduced uniform-remesh quadrangulate; it does not implement the promised voxel remesh or `surfaceDistance` semantics. UE 5.8 exposes explicit edge-length versus approximate triangle-count modes, and its remesh implementation passes the selected edge length through to the remesh op. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\AdvancedMeshOpsHandler.cpp:3-4`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshOpsHandler.cpp:1440-1481`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Tests\Geometry\TestGeometryRemeshUniformEchoesMeshCounts.cpp:45-47,92-116`, `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Public\GeometryScript\MeshRemeshFunctions.h:17-22,133-141`, `C:\UE_5.8\Engine\Plugins\Runtime\GeometryScripting\Source\GeometryScriptingCore\Private\MeshRemeshFunctions.cpp:76-83`. Runtime: NOT VERIFIED. Recommendation: CLOSE CANDIDATE; do not resurrect these RPCs from this ticket; if a retopology capability is wanted, file a new contract defining quadrangulation versus voxel behavior and add current UE 5.8 differential tests before implementation.
- `#6-additional-adversarial-removal-check` `OPEN` reporter — Additional evidence: **Adversarial review B — CLOSE CANDIDATE.** Actuality: STALE/FIXED. Framing: I could not falsify A: nested PinWright history commit `e0d0fe2c6297483ee78f305d6c4421ec13207143` deletes both registrations and bodies; current post-split source has no `geometry.quadrangulate`, `geometry.remesh_voxel`, or `targetQuadSize`, and the replacement record maps them to `geometry.remesh_uniform` (plus `geometry.fill_holes` for voxel behavior). There is no surviving alias that can still ignore the old knobs. I agree with A's stale/fixed result; no substantive disagreement. Proposed fix: NONE, the old handlers are gone; the abandoned TargetEdgeLength patch cannot apply to a live API, and reintroducing retopology needs a new contract. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\AdvancedMeshOpsHandler.cpp:3-4` `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWrightGeometry\Private\Handlers\Geometry\MeshOpsHandler.cpp:1440-1462` `X:\src\unreal\.pinwright-board\E-rpc-cull-151-record.md:169-173`. Runtime: NOT VERIFIED. Recommendation: CLOSE CANDIDATE; leave this historical ticket unchanged and file a new replacement-capability ticket with UE 5.8 differential tests if needed.

- `#2-verbs-deleted-guard-added` `IN-REVIEW` developer — Resolves as **already removed, plus a guard on the successor**. Both named verbs are gone from current source: `grep -rni 'quadrangulate|remesh_voxel'` over the whole plugin returns nothing, and `git log -S` puts their deletion in e0d0fe2c 'RPC audit: remove 170 dead methods'. The shared `FMath::Max(100, TrisBefore / 2)` pattern the ticket used to enumerate them returns zero hits across Source/. So there was no parameter left to implement or reject. What was left was the class: `geometry.remesh_uniform` is the retopology verb that survived, it does read `targetTriangleCount`/`targetEdgeLength` (MeshOpsHandler.cpp:1738-1800), and nothing failed if that regressed — the existing `.EchoMeshCounts` test asserts only that the response carries a non-zero achieved count, which a hardcoded density satisfies perfectly. Added `PinWright.geometry.remesh_uniform.DensityRequestChangesTheOutput` (Tests/Geometry/TestGeometryRemeshUniformHonoursDensity.cpp): the same box fixture at 200 and 6000 triangles, asserting the achieved counts differ in the requested direction. It asserts order rather than value because uniform remesh only approximates a budget; a density independent of the request converges the two runs and fails it. Commit b4f5bd58. Compile-checked -SingleFile, [1/1] Compile, Result: Succeeded.
