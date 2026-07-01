---
id: F-crir-template-and-dispatch
title: "CRIR — template and dispatch node coverage"
status: DONE
severity: Low
category: feature
tags: [control-rig, rigvm, ir, crir, opcodes, template, dispatch]
---

# CRIR — template and dispatch node coverage

Carved out from `F-crir-extended-node-kinds` during the wave-1 sprint. Wave 1
shipped the `if` and `select` opcodes, which map to `URigVMDispatchNode`
instances dispatched through the `FRigVMDispatch_If` /
`FRigVMDispatch_SelectInt32` factories. The remaining template/dispatch
coverage is:

- Bare `URigVMTemplateNode` instances created via
  `URigVMController::AddTemplateNode(FName Notation, ...)`. These represent
  unresolved type-parameterized RigVM functions and don't fit the existing
  `unit` opcode shape (which keys off a struct path).
- `URigVMDispatchNode` instances *other than* if/select — math, array, and
  other factory-dispatched kinds outside the wave-1 set. The decompiler
  currently emits them as `# TODO unsupported node kind: RigVMDispatchNode`.

Note that `URigVMUnitNode : URigVMTemplateNode`, so the existing wave-1
`unit` opcode already covers the most common template shape (a resolved unit
with a script-struct path). This ticket is about the *bare* template case
plus the broader dispatch fan-out.

## Resolved design decisions

- **Bare template disambiguation:** introduce a new top-level `template` opcode
  keyed off `Notation` (FName). Reasoning: wave 1's `unit` opcode grammar-locks
  on a `/Script/Path.RigUnit_X` token shape; bare templates have a notation
  like `MyTemplate(in T,out T)` that does not parse as a struct path, so
  reusing `unit` would force grammar lookahead and break the one-line/one-shape
  precedent established by BPIR/AGIR. Distinct opcode is one fewer rule for
  LLM write-paths.
- **Dispatch factory coverage:** single discriminated `dispatch <FactoryStructName>`
  opcode for all factories except `if`/`select` (already shipped as sugar).
  The decompiler at `CRIRDecompiler.cpp:419` already discriminates via
  `Factory->GetScriptStruct()->GetName()` — the same token works as the
  write-path key. Reject per-factory sugar opcodes (`array_add`, `branch`, …);
  grammar bloat without LLM/MCP benefit.

**Fix:** Add a `template` opcode that calls
`URigVMController::AddTemplateNode(FName Notation, ...)`. Add a `dispatch`
opcode taking the factory script-struct name as its discriminator and routing
through the matching `URigVMController::Add*` helper (or `AddTemplateNode`
keyed on the factory's notation when no dedicated helper exists). Decompiler
classifies `URigVMTemplateNode` (bare) vs `URigVMDispatchNode` (factory) vs
`URigVMUnitNode` (resolved unit) and picks the matching opcode.

## Depends on

- `F-crir-extended-node-kinds` (wave 1) must be in place — this ticket
  reuses the if/select dispatch-node Cast and factory-name discrimination
  established there.

## History
- `#1-initial-spec` `OPEN` developer — Carved out from F-crir-extended-node-kinds wave 1. Phase A's unit opcode already covers URigVMUnitNode which inherits URigVMTemplateNode; adding a separate template opcode requires deciding how to disambiguate bare templates from units. Includes URigVMDispatchNode coverage with its own creation API.
- `#2-design-decisions-resolved` `OPEN` developer — Resolved both open design questions during board-grooming session. (1) Use a distinct `template` opcode rather than overloading `unit` — grammar precedent and LLM clarity. (2) Use a single discriminated `dispatch <FactoryStructName>` opcode for non-if/select; reject per-factory sugar. Ticket is now sprint-ready.
- `#3-template-and-dispatch-opcodes` `IN-REVIEW` developer — Added `template` and `dispatch` opcodes. Compiler routes both through `AddTemplateNode(Factory->GetTemplateNotation())` uniformly — skipped per-factory helper branch as YAGNI per analysis. Decompiler classifier extended: pre-pass lambda broadened, bare-template arm inserted after unit arm, general dispatch arm inserted after if/select special case. Regression tests in TestCRIRTemplateAndDispatch.cpp cover bare template, branch dispatch, array-add dispatch.
- `#4-verify-fix` `DONE` tester — Verified: ran the three regression tests via `system.run_tests` with `tests=["EditorAutomationRpcGateway.CRIR.RoundTrip.Template","...DispatchPrint","...DispatchArrayAdd"]` (job j_20260518T092421_9da73422). All three resolved and completed in 14s with `has_errors: false` and `missingTests: []`, confirming the template/dispatch opcodes round-trip cleanly for bare templates, Print dispatch, and ArrayAdd dispatch.
