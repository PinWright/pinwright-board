---
id: B-undo-last-bpir-doesnt-restore-phase0-sweeps
title: "blueprint.undo_last_bpir doesn't restore nodes that the compile's Phase 0 swept"
status: DONE
severity: Medium
category: bug
tags: [bpir, undo, rollback]
---

# blueprint.undo_last_bpir doesn't restore nodes that the compile's Phase 0 swept

`blueprint.undo_last_bpir` deletes nodes created by the most recent BPIR compile and returns `{deletedCount: N}`, but does NOT restore nodes that the same compile's Phase 0 deletion BFS swept (custom event bodies, shared subgraph nodes reachable from a re-declared entry). The forward direction deletes-then-creates; the undo direction only un-creates. A user expecting a true rollback to the pre-compile state is left with the swept nodes still missing.

Root cause (verified in source):
- `BpirCompilerHandler.cpp:213-223` — RPC `mode` default `"append"` (and the legacy `"replace"` alias) maps to `EBpirCompileMode::Replace`, which **does** run Phase 0 deletion. Only `"extend"` skips it.
- `BpirCompiler.cpp:2040-2061` — comment explicitly documents the asymmetry: "the atomic rollback at the end of Compile() only removes nodes CREATED during the compile (tracked via NodeEmitter GUIDs); it cannot restore nodes deleted in Phase 0. UndoCompile: PopLastCreatedNodes + DeleteNodesByGUIDs must return to baseline, which also relies on no Phase 0 deletions in Default mode." The precondition is violated for the normal RPC path because the handler maps the default RPC mode to `Replace`, not `Default`.
- `BpirCompilerHandler.cpp:739-754` (`undo_last_bpir`) — only calls `FBpirCompiler::PopLastCreatedNodes()` + `DeleteNodesByGUIDs`. No counterpart `PopLastDeletedNodes` exists; nothing serializes Phase-0-swept node payloads for restoration.

Distinct from `B-no-undo-redo` (DONE) and `B-compile-bpir-transaction-ensure` (DONE): both addressed transaction wrapping and the failure-path rollback inside a single compile. This ticket is about the **successful** compile + later explicit `undo_last_bpir` round-trip leaving the asset in a non-pre-compile state.

**Possible fixes:**
1. Snapshot the export text of Phase-0-swept nodes (and their pin links) into a per-compile undo record alongside `NodeCreationStack`, then re-import them in `undo_last_bpir` before deleting the created nodes.
2. Or: refuse `undo_last_bpir` when the recorded compile ran Phase 0 sweeps, returning a structured `UNDO_NOT_REVERSIBLE` error so callers fall back to git/asset.revert instead of getting a half-rollback.
3. Or: route `undo_last_bpir` through the editor transaction buffer (`GEditor->UndoTransaction`) on a transaction record that captured the full pre-compile state — same machinery as `editor.undo`.

## History
- `#1-initial-repro` `OPEN` reporter — On `/App/App/UI/W_FoundGasLeaks`: `compile_bpir` with a new `entry event Tick(...)` body returned `{nodeCount: 10, success: true}` but also silently emptied `OnSearchStateChanged_Event` body via Phase 0 BFS. `blueprint.undo_last_bpir` returned `{success: true, deletedCount: 20}`. `blueprint.decompile` after undo: Tick body partially gone (cast demoted to pure form, no @ok branch, orphan Set Text), AND `OnSearchStateChanged_Event` still empty — NOT restored. Net: undo + previous compile leaves the asset more broken than either alone. Compiler source comment (`BpirCompiler.cpp:2040-2061`) acknowledges the asymmetry as a known limitation when default RPC mode runs Phase 0.
- `#2-undo-not-reversible-error` `IN-REVIEW` developer — Added `Phase0RanStack` parallel to `NodeCreationStack` in `FPluginState`; `undo_last_bpir` in `BpirCompilerHandler.cpp` now returns structured `UNDO_NOT_REVERSIBLE` error when the most-recent compile ran Phase 0 sweeps. Callers fall back to git / asset.revert. Handler test added at `TestBpirUndoLastBpir.cpp` covering both Replace (refused) and non-Replace (succeeds) paths.
- `#3-verified-undo-refused-on-replace` `DONE` tester — Verified: `mcp__editor-automation__call` with `path: "blueprint.undo_last_bpir"`, `args: {assetPath: "/App/App/UI/W_FoundGasLeaks"}` after a Replace-mode `compile_bpir` returned the structured error `[UNDO_NOT_REVERSIBLE] undo_last_bpir cannot reverse Phase 0 sweeps from the previous compile. The compile ran in append/replace mode and deleted pre-existing entry subgraphs that aren't snapshotted for restoration. Use git revert or asset.revert instead.` exactly matching the IN-REVIEW description. Pre-fix the same call returned `success: true, deletedCount: 20` with the asset still corrupted. Behaviour now correctly refuses rather than half-rolling-back; caller is directed to git / asset.revert. Note: ticket text references `asset.revert` as the suggested alternative, but no `asset.revert` RPC exists in the `asset.*` namespace — the practical fallback is editor-side asset reload or `git checkout` on the on-disk `.uasset`. Worth filing a follow-up to wire `asset.revert` (or update the error message to drop the unimplemented suggestion).
