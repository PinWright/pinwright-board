---
id: E-asset-dump-ir-sidecar-registry
title: "Move text IR asset-dump sidecars into a registry"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, refactor, ir-sidecar, merge-conflict-surface]
---

# Move text IR asset-dump sidecars into a registry

`AssetDumpHandler.cpp` owns the asset dump orchestration and should keep
type-specific, non-uniform sidecars explicit. The current text IR sidecars
for MGIR, AGIR, SCIR, and BTIR all share the same dump-time shape:
decompile an asset into text, record warnings with the existing asset-dump
diagnostic path on failure, and write one canonical `*.txt` sidecar on
success.

Centralizing only that uniform text IR dispatch in a small registry reduces
the merge-conflict surface when adding another text IR sidecar. BPIR remains
explicit because `AssetDumpBuilder::BuildBpirText` returns plain dump-specific
text with graph headers and failure markers, not a uniform `{ Text, Warnings,
bSuccess }` result. JSON builders and other one-off sidecars also remain
explicit in `AssetDumpHandler.cpp`.

**Scope:** registry for MGIR, AGIR, SCIR, and BTIR text sidecars only. The
registry stores class discriminators as `UClass* (*)()` thunks, sorts specs by
priority and stable filename/name order, and writes at most one sidecar for a
filename after the highest-priority matching spec handles it.

**Discoverability:** each IR decompile handler registers its dump sidecar with
a loud `REGISTER_DECOMPILE_IR` macro near the corresponding RPC handler, so
the dispatch table remains easy to find with text search.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from architecture audit.
  Confirmed evidence: `AssetDumpHandler.cpp` is 1,815 LoC, 23 `IsA<>`
  /`Cast<>` hits in the file (most clustered in `BuildAllFilesForAsset`
  ~424–686). Three IR helpers (BPIR/MGIR/AGIR) are ~16-line
  near-duplicates. No existing registry/sidecar-refactor ticket on the
  board. Marked NON-BLOCKING for new IRs.
- `#2-reformulated-current-scope` `OPEN` developer — Renamed invalid
  `R-asset-dump-sidecar-registry` to `E-asset-dump-ir-sidecar-registry`
  because `R-*` is not a valid board prefix and this is ergonomic refactor
  work. Narrowed the scope to current text IR sidecars only: MGIR, AGIR,
  SCIR, and BTIR. BPIR and non-text/non-uniform sidecars stay explicit in
  `AssetDumpHandler.cpp`.
- `#3-registered-text-ir-sidecars` `IN-REVIEW` developer — Added
  `Utils/IrSidecarRegistry.{h,cpp}` with thunk-based class discriminators,
  priority plus filename/name sorting, and `REGISTER_DECOMPILE_IR`
  registrations in the MGIR, AGIR, SCIR, and BTIR decompile handlers.
  Updated `AssetDumpHandler.cpp` to dispatch registered text IR sidecars
  through the existing diagnostic/write pattern and to load registered IR
  filenames dynamically for diff baselines. No new tests were required; the
  existing `mgir.txt`, `agir.txt`, `scir.txt`, and `btir.txt` dump coverage
  remains the regression surface.
- `#4-verify-registry-sidecar` `DONE` tester — Verified: live `asset.dump` on `/App/App/Drone/Mini/Materials/M_Opaque_Master1` emitted `mgir.txt` through the registered text-IR sidecar path, and the dumped file starts with `entry material` for the sampled material. Static source check confirms `REGISTER_DECOMPILE_IR` registrations for MGIR, AGIR, SCIR, and BTIR plus `AssetDumpHandler.cpp` dispatch through `IrSidecarRegistry::GetRegisteredIrSidecars()`.
