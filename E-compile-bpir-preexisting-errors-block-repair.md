---
id: E-compile-bpir-preexisting-errors-block-repair
title: "compile_bpir rolls back valid placement when OTHER graphs carry pre-existing compile errors — blocks incremental repair of a broken BP"
status: OPEN
severity: Medium
category: ergonomic
tags: [bpir, compile_bpir, rollback, whole-bp-validation, repair]
---

# compile_bpir rolls back valid placement when OTHER graphs carry pre-existing compile errors

After a successful BPIR placement, `compile_bpir` runs a **whole-Blueprint** compile
(`CompileBlueprintWithDiagnostics(BP)`, `BpirCompilerHandler.cpp:283`) which calls
`FKismetEditorUtilities::CompileBlueprint` on the entire BP and reads `Blueprint->Status`
(`BlueprintHandlerUtils.cpp:407-445`). If ANY untouched graph is broken, the BP goes
`BS_Error`, `bCompiled=false`, and the handler rolls the placement back
(`:318-322` → `bRollback=true` → `:349-352`). So when a BP is already broken — e.g. a
C++ struct rename orphaned five graph regions — fixing ONE graph fails because the four
untouched graphs still error. Worse, the error string joins ALL `Diagnostics.Errors` with
`"; "` (`BuildBlueprintCompileFailureMessage`, `:150-164`) with no marker for which are
pre-existing, so the caller sees errors from graphs it never touched and its good placement
silently discarded. There is no baseline compile before placement to diff against.

**Workaround:** Pre-delete every broken node in the untouched graphs via
`blueprint.graph.delete_node`, then upsert ALL corrected entries in ONE `compile_bpir`
call so the whole BP compiles clean in a single pass. Works, but is undiscoverable — the
wiki's "Failed-compile rollback" note documents the mechanism, not this repair implication.

**Fix:** Compile the BP once before placement to capture a baseline error set, diff against
the post-placement compile, and only roll back when NEW errors appear; or add an
`allowPreexistingErrors` flag that keeps the placement when the failing errors are unchanged
from baseline. At minimum, add a `blueprint.bpir-gotchas.md` bullet prescribing the
pre-delete + all-in-one-call repair pattern.

## History
- `#1-initial-repro` `OPEN` reporter — Repairing `/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone` (5 graph regions referencing a deleted C++ struct): a `compile_bpir` with a single corrected `entry function RefreshPhysicsPanel(){...}` returned `[BLUEPRINT_COMPILE_FAILED] ... In use pin 'Drone Physics Settings' no longer exists on node 'Make <unknown struct>'; ... 'Physics' no longer exists on node 'Break Drone Data'...` — every listed error was from OTHER graphs (Reset/Copy/Paste handlers, SavePhysicsSettingsIntoDrone) untouched by the call; the RefreshPhysicsPanel placement was valid and got rolled back. Verified in source: whole-BP compile at `BpirCompilerHandler.cpp:283` (`CompileBlueprintWithDiagnostics` → `BlueprintHandlerUtils.cpp:407-445`, reads `Blueprint->Status`, no baseline), rollback gated on `!Diagnostics.bCompiled` (`:318-322`, `:349-352`), error string joins all diagnostics with no pre-existing marker (`:150-164`). Workaround used: deleted 7 broken pure nodes via `graph.delete_node`, then one `compile_bpir` with all five entries → success (42 nodes, compiled:true). Refinement of `E-bpir-auto-validate` (DONE, added the auto-validation) + `B-false-compile-success` (DONE, why it was added) — not a reopen. Severity set Medium (not the suggested High): the all-in-one workaround works reliably, so it is a soft blocker on a rare struct-rename repair path, not a hard blocker.
