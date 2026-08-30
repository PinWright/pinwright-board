---
id: E-agir-roundtrip-symbol-not-found-undiagnosable
title: "AGIR round-trip `AGIR_SYMBOL_NOT_FOUND` is undiagnosable from the tool surface — forces reading plugin C++ source"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [agir, animgraph, roundtrip, error-message, discoverability, diagnosability, docs]
---

# AGIR round-trip failures are undiagnosable without reading plugin source

When `anim.compile_agir` rejects the **verbatim output of
`anim.decompile_agir`** with a pose-symbol error, the message is locally
true but globally misleading about the real cause:

```
[AGIR_SYMBOL_NOT_FOUND] Pose reference '%<token>' did not resolve
to a UAnimGraphNode_Base (line N).
```

A caller reading this reasonably concludes *their AGIR text is malformed* —
that they referenced a symbol they never defined. But when the text was
emitted **by the decompiler itself, unmodified**, the real cause is a
decompiler/compiler round-trip asymmetry (the emitter writes a `%`-pose-ref
the compiler can't reconstruct), not a user typo. The generic pose-link
resolver — `AGIRCompiler.cpp` `ResolvePendingPoseWires` (the
`AGIR_SYMBOL_NOT_FOUND` site at lines ~1135-1141) — has **no way to tell the
two apart** and points the caller at *line N of their input* either way.

## Why this is process friction (independent of any single round-trip bug)

This is the **diagnosability** angle, which outlives any single round-trip
defect. It is deliberately *not* about one repro:

- There is **no signal on the tool surface** that "this is decompiler
  output the compiler can't reconsume" vs. "you wrote bad AGIR." The error
  text is identical for both.
- The canonical AGIR workflow is *decompile → edit text → recompile → diff*.
  When recompile of unmodified output fails, the caller has **no
  tool-level way to tell whether they broke it or the tool did** — so the
  only path to a diagnosis is reading the plugin's C++
  (`AGIRTextEmitter` / `AGIRParser` / `AGIRCompiler`). The audited task did
  exactly this and flagged it: *"I had to read plugin AGIR source
  (AGIRTextEmitter/AGIRParser/AGIRCompiler — last resort, a real
  discoverability gap) to diagnose it, then pivot to a different AnimBP."*
  Reading plugin source to use an MCP tool is the friction.
- The gap is **structural, not tied to the one defect that motivated this
  ticket.** The `%state_machine_0` round-trip break that originally
  surfaced it (`B-agir-state-machine-output-pose-unbound`, now IN-REVIEW
  with a fix) is closed, but the generic resolver still fires hint-less for
  *any* future unresolved pose ref, and there is at least one still-OPEN
  lossy round-trip case (`B-agir-alphaboolblend-empty-struct-not-reimportable`,
  empty-struct `AlphaBoolBlend: "()"`) where unmodified decompiler output is
  not losslessly re-consumable. The friction recurs on every such case.

## What it should do

Two complementary improvements, either of which removes the source-reading:

1. **Better error text / hint on the tool surface.** When
   `AGIR_SYMBOL_NOT_FOUND` fires on a `%`-pose-ref during
   `ResolvePendingPoseWires`, attach a **generic** round-trip diagnostic
   hint to the compile result — not a fragile per-case heuristic tied to one
   emitter path (the original sketch keyed on the now-fixed
   `output %state_machine_<n>` pre-allocation; that specific path no longer
   fails, so the hint must be class-level, true for any unresolved pose ref).
   `anim.compile_agir` then relays that hint on the error response. Even a
   generic
   `hint: "if this %-pose-ref came from unmodified anim.decompile_agir
   output, this is likely a tool round-trip gap, not your edit — see the
   anim wiki 'Round-trip limitations' section before rewriting your AGIR"`
   redirects the caller away from re-reading their own text.

2. **Document the round-trip caveats in the wiki.** The `anim` wiki page
   (`Docs/wiki-src/anim.md`) documents the *cliff families that DO
   round-trip* and the field-name conventions, but has **no "known
   round-trip limitations / how to diagnose a round-trip failure" section**.
   It should name the currently-known not-yet-lossless case (empty-struct
   fields like `AlphaBoolBlend: "()"` — see
   `B-agir-alphaboolblend-empty-struct-not-reimportable`), note that the
   `%state_machine_0` single-top-level-state-machine case was a historical
   instance now fixed, and state the diagnostic rule explicitly: *"If
   `anim.compile_agir` rejects the unmodified output of
   `anim.decompile_agir`, that is a tool round-trip gap, not your error —
   check this list / file a board ticket rather than rewriting your AGIR."*
   That single overlay paragraph short-circuits the source-reading detour.

## Evidence (historical — original repro now fixed)

The motivating audit was an `anim` AGIR round-trip task whose first asset
pick (a state-machine AnimBP, `2-5_ABP_StateMachines`) failed to round-trip
its own decompiler output: `anim.compile_agir` returned
`[AGIR_SYMBOL_NOT_FOUND] Pose reference '%state_machine_0' did not
resolve...` in **both Replace and default mode** (2 failed compile calls).
The friction note: *"AGIRTextEmitter/AGIRParser/AGIRCompiler — last resort, a
real discoverability gap — to diagnose it."* The caller then had to abandon
the chosen asset and scout/pivot to a non-state-machine AnimBP (3 extra
decompile "scout" calls + `asset.duplicate`) before any productive edit.

**This specific `%state_machine_0` repro no longer reproduces** — the
source-level fix is present (`B-agir-state-machine-output-pose-unbound`,
IN-REVIEW: the compiler now registers the name-token state machine under the
emitter's positional id). But the diagnosability gap it exposed is durable:
the generic resolver still gives no attribution signal for any unresolved
pose ref, and the still-OPEN `B-agir-alphaboolblend-empty-struct-not-reimportable`
is a live lossy round-trip case. A clean authoring intent still risks a
source-reading detour purely because the tool surface gives no way to
attribute the failure.

**Wiki page to improve (downstream wiki process):**
`Docs/wiki-src/anim.md` — add a "Round-trip limitations & how to diagnose a
round-trip failure" section listing the known not-yet-lossless case
(`AlphaBoolBlend: "()"`) and the "tool gap, not your error" diagnostic rule.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from a struggle audit of an
  `anim` AGIR round-trip task. The diagnosability/discoverability angle is
  distinct from the underlying defect `B-agir-state-machine-output-pose-unbound`
  (which fixes the one symbol-binding bug): `AGIR_SYMBOL_NOT_FOUND` on
  decompiler-emitted text gives the caller no way to tell "tool round-trip
  gap" from "my malformed AGIR," forcing a read of plugin C++
  (`AGIRTextEmitter`/`AGIRParser`/`AGIRCompiler`) to diagnose — the task's
  friction note calls this "a real discoverability gap." Evidence: 2 failed
  `anim.compile_agir` calls (Replace + default mode) on `2-5_ABP_StateMachines`
  plus a 4-call scout/pivot to a non-state-machine AnimBP. Proposed fix:
  (a) a compile-side hint distinguishing decompiler-output round-trip gaps
  from user errors, and (b) a "round-trip limitations / how to diagnose"
  section on `docs/wiki-src/anim.md`.
- `#2-reword-and-implement` `IN-REVIEW` developer — Reworded to reality: the
  named `%state_machine_0` repro is now fixed (`B-agir-state-machine-output-pose-unbound`,
  IN-REVIEW), so the lead exhibit was re-anchored on the durable resolver-level
  diagnosability gap and the still-OPEN
  `B-agir-alphaboolblend-empty-struct-not-reimportable` lossy case; the
  state-machine repro is marked historical in Evidence; proposal #1 was
  generalized from a fragile per-emitter-path heuristic to a class-level hint
  (true for any unresolved pose ref). Implemented both remedies. (1) Compile-side
  hint: added `FString Hint` + `WithHint()` to `FAGIRCompileResult`
  (`Source/EditorAutomationRpcGateway/Private/AGIR/AGIRCompiler.h`); the
  `AGIR_SYMBOL_NOT_FOUND` site in `ResolvePendingPoseWires`
  (`Source/EditorAutomationRpcGateway/Private/AGIR/AGIRCompiler.cpp`) now attaches
  a generic round-trip diagnostic hint (file-scope `GAGIRPoseSymbolNotFoundHint`);
  the hint propagates unchanged through `FAGIRCompiler::Compile`'s block-error
  return path, and `anim.compile_agir`
  (`Source/EditorAutomationRpcGateway/Private/Handlers/Animation/AGIRCompileHandler.cpp`)
  relays it as a `hint` field on the error's structured result via the 3-arg
  `Ctx.SendError`. (2) Wiki: added a "Round-trip limitations & how to diagnose a
  round-trip failure" section to `Docs/wiki-src/anim.md` with the "tool gap, not
  your error" diagnostic rule, naming the OPEN AlphaBoolBlend case and noting the
  state-machine case as the historical (now-fixed) example. Regression test:
  `FAGIRSymbolNotFoundCarriesRoundTripHintTest`
  (`EditorAutomationRpcGateway.anim.agir.SymbolNotFoundCarriesRoundTripHint`) in
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAnimGraphHandlers.cpp`
  — compiles AGIR text whose `output` references an unbound `%`-pose-ref against a
  fresh valid target, asserts the result fails with `AGIR_SYMBOL_NOT_FOUND` and
  carries a non-empty round-trip diagnostic Hint; fails if the `.WithHint(...)`
  attach is reverted. Did not compile/run tests (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 4 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
