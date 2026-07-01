---
id: F-ir-grammar-harmonize
title: "Harmonize BPIR/MGIR/AGIR text grammars before SCIR/BTIR/MSIR/NIR lock in 5-way divergence"
status: WONTFIX
severity: High
category: feature
tags: [ircore, bpir, mgir, agir, refactor, grammar, prereq]
---

# Harmonize IR text grammars before the next four IRs land

Three IRs ship today (BPIR, MGIR, AGIR) and four more are tracked on this
board (state-tree / behavior-tree / metasound / niagara — see `F-control-rig-ir-language`,
`F-state-tree-conditions-evaluators`, `F-niagara-graph-create-node`,
`F-bpir-animgraph-not-walked` plus the upcoming chooser/PCG IR work). Today
the three current grammars diverge on five low-level surface details. Each
new IR added without harmonizing first multiplies the divergence and the
post-hoc fix cost. Land this before any further IR-text emitter is written.

Sister ticket `F-ircore-shared-text-helpers` already consolidated the
*helpers* (escape, quote, position, name-token) into `IrCore/IrTextUtils`.
What it didn't do is force every IR to *use them consistently*, and it
didn't address the four divergences below that live above the helper layer.

## Verified inconsistencies

All five confirmed by direct source inspection.

### 1. Field separator: `=` vs `: ` (AGIRTextEmitter)
- `Source/.../AGIR/AGIRTextEmitter.cpp:388, 433, 465, 710` hardcode
  `TEXT("%s=%s")` for `Key=value` field pairs.
- `Source/.../MGIR/MGIRDecompiler.cpp:436` uses `TEXT("ConstCoordinate: %d")` —
  colon, space-padded. BPIR's `BpirTextEmitter.cpp:587` also uses
  `TEXT("%s: %s")`.
- Fix: emit `Key: Value` everywhere. Change emitter and parser together
  (IR text is ephemeral — no storage, no migration window).

### 2. Entry-name encoding: double-quoted vs backtick/bare
- `MGIRDecompiler.cpp:861, 888` hardcode
  `Printf(TEXT("entry material %s {"), *FIrTextUtils::Quote(Material->GetPathName()))` —
  double-quoted asset path.
- AGIR (`AGIRTextEmitter.cpp:269`) and BPIR route through
  `FIrTextUtils::FormatNameToken` which emits bare token or backtick-wrapped
  when the token contains delimiters.
- Asset paths contain `/` and `.` so they require quoting/wrapping; the
  canonical wrapping is backtick per `F-ircore-shared-text-helpers` #2.
- Fix: replace `Quote(GetPathName())` with `FormatNameToken(GetPathName())`
  at both MGIR sites.

### 3. AGIR register naming: opaque vs class-mnemonic
- `AGIRTextEmitter::MakeLocalId` (line 221) returns
  `FString::Printf(TEXT("%%n%d"), Counter)` — opaque `%n0, %n1, %n2, …`.
- BPIR uses mnemonic suffixes when it can; MGIR uses
  `n<guid12>` (line 55). The proposal: derive from class short-name —
  e.g. `%state_machine_0`, `%blend_space_0`, `%save_cached_pose_0`.
- Recompile is name-blind (the compiler binds nodes by graph position, not
  by token spelling), so the migration is zero-risk for round-trip identity.

### 4. MGIR baked-in indent
- `MGIRDecompiler.cpp:454, 460, 467, 471, 479, 482` carry literal
  `TEXT("    %%%s = constant Float1(...)")` — 4-space prefix inside the
  Printf format string.
- AGIR (line 281) and BPIR keep indent out of the format string and join
  with `Lines.Add(TEXT("    ") + Line)`.
- Fix: pull MGIR's emit functions to return un-indented strings, prepend
  indent at the join site. Post-processors (formatters, syntax highlighters)
  can then control indent uniformly.

### 5. Class-name qualification
- AGIR (`AGIRTextEmitter.cpp:486`) emits
  `Node->GetClass()->GetPathName()` — fully qualified
  `/Script/AnimGraph.AnimGraphNode_StateMachine`.
- MGIR (`MGIRDecompiler.cpp:120`) strips `MaterialExpression` prefix and
  emits bare `Multiply`, `TextureSample`, `Constant3Vector`.
- The 4 upcoming IRs (state-tree, behavior-tree, metasound, niagara) all
  have open class sets where short-name collisions are guaranteed. Prefer
  AGIR's fully-qualified convention.
- Fix: switch MGIR to qualified emission. Provide a parser-side short-name
  alias only for the unambiguous `MaterialExpression*` set if humans need it.

## Proposed harmonized core

Every IR (current and upcoming) agrees on this surface:

| Element | Form | Example |
|---------|------|---------|
| Entry header | `entry <kind> <name-token> { … }` | `entry material `/Game/Mat/M_Foo` {` |
| Comment | `#` (quote-aware) | `# top-level constant` |
| Position annotation | `@(x, y)` | `@(128, -64)` |
| Field list | `(Key: value, Key: value)` | `(ConstCoordinate: 0, SamplerSource: SSM_FromTextureAsset)` |
| String literal | double-quoted, standard escapes | `"hello \"world\""` |
| Diagnostic shape | `{Line, Column, Code, Message}` | `{12, 4, AGIR_UNKNOWN_NODE, "Unrecognized node …"}` |
| Diagnostic code namespacing | `<IR>_<UPPER_SNAKE>` | `MGIR_DANGLING_INPUT`, `BPIR_MISSING_EXEC` |
| Register naming | `%<class_mnemonic>_<n>` | `%blend_space_0`, `%texture_sample_2` |

## Migration risk

IR text is ephemeral (asset-dump sidecars + transient compile inputs). No
backwards-compat needed; just change emitter and parser together.

## Why this is a hard prerequisite for the next four IRs

If SCIR / BTIR / MSIR / NIR each get authored against the *current* state,
the post-hoc harmonization touches 7 emitters × 5 axes = 35 sites, every
parser gets a dual-accept window per axis, every IR's downstream tooling
(formatters, syntax highlighters, AI scaffolds, MCP wiki rendering) needs
five branches. The cost of NOT fixing first scales linearly with each new
IR added; 5-way grammar lock-in is genuinely expensive to undo.

Fixing first means the four new IRs are written against one grammar and
ship harmonized by construction.

## Cross-references

- `F-ircore-shared-text-helpers` (DONE) — helper-layer consolidation. This
  ticket sits on top: forces the *call sites* of those helpers to converge.
- `F-mgir-material-graph-ir` (parent for axes 2, 4, 5).
- `F-control-rig-ir-language` (upcoming IR; must inherit harmonized grammar).
- `F-bpir-animgraph-not-walked` (related anim-side work).
- `F-state-tree-conditions-evaluators`, `F-niagara-graph-create-node`,
  `F-chooser-namespace` (likely upcoming-IR carriers).
- `E-mandatory-node-position` (overlaps axis: `@(x, y)` annotation form).
- `E-bpir-magic-strings-and-duplications` (overlaps motivation: prevent
  per-IR drift).

## Approach

Five independent passes, each a small reviewable diff:

1. **AGIR field separator** — replace `"%s=%s"` with `"%s: %s"` at four
   call sites in `AGIRTextEmitter.cpp`. Update `AGIRParser` to expect
   `: ` only. Regenerate fixture text.
2. **MGIR entry header** — swap `Quote` → `FormatNameToken` at lines 861,
   888. Update `MGIRParser` to expect backtick-wrapped tokens only.
3. **AGIR register naming** — replace `MakeLocalId` body with
   class-mnemonic lookup keyed on `Node->GetClass()`. No parser change
   (binding is positional).
4. **MGIR indent extraction** — strip leading `"    "` from MGIR Printf
   formats; prepend at the join site.
5. **MGIR class qualification** — switch `ExpressionClassName` to emit
   `GetClass()->GetPathName()`. `MGIRParser` accepts only qualified form.

Each pass: regenerate round-trip fixtures against the new grammar.

**Fix:** implement the five harmonization passes with these clarifications: AGIR parenthesized field lists become `Key: value` and reject top-level `=` in those lists; MGIR entry headers emit `FIrTextUtils::FormatNameToken(GetPathName())` and `FMGIRParser` accepts name tokens only, not double-quoted strings; AGIR register ids use node/opcode class mnemonics with per-mnemonic counters, updating definitions and references together; MGIR indentation is centralized at body append sites for all body lines, not only constants; MGIR call class names emit qualified `GetClass()->GetPathName()` name tokens and MGIR parsing/compile rejects short `call` class names while leaving `FMaterialExpressionFactory` aliases available to non-MGIR callers. Update tests and `Docs/wiki/anim.md` / `Docs/wiki/material.mgir.md`. No dual-accept parser window.

## History
- `#1-initial-spec` `OPEN` reporter — Filed during 4-IR pre-landing audit.
  Verified all five inconsistencies by direct source inspection
  (`AGIRTextEmitter.cpp` lines 221, 388, 433, 465, 486, 710;
  `MGIRDecompiler.cpp` lines 120, 436, 454-482, 861, 888). No existing
  harmonization ticket on the board. Hard prereq for the four upcoming IRs
  (state-tree, behavior-tree, metasound, niagara) because cost of post-hoc
  harmonization scales with IR count. `F-ircore-shared-text-helpers`
  consolidated the helpers but did not converge the call sites; this
  ticket finishes the job above that layer.
- `#2-no-backcompat-needed` `OPEN` reporter — Clarifying project-level
  invariant: **IR text (BPIR, MGIR, AGIR, and any future IR like
  SCIR/BTIR/MSIR/NIR) is ephemeral.** It exists only as (a) asset-dump
  sidecars rewritten on every re-dump, or (b) compile-input strings
  immediately converted to assets and discarded. There are no users
  storing IR text. Consequences for this ticket: dual-accept parser
  windows and deprecation cycles called out in earlier wording are
  explicitly NOT NEEDED; no backcompat shims; no migration risk. Proposed
  Fix simplifies to: emit and parse change together in one pass per axis;
  existing tests pinning the old format regenerate alongside the grammar
  change (no "preserve old behavior" tests retained). Body sections
  updated: Migration risk reduced to a one-liner, dual-accept/back-compat
  language struck from axis #1, axis #2, axis #5, and the Approach
  passes.
- `#3-harmonized-ir-grammar` `IN-REVIEW` developer — Implemented AGIR colon field-list parsing/emission and mnemonic local ids, MGIR name-token entry headers, qualified MGIR call class tokens, centralized MGIR body indentation, stale MGIR/AGIR regression updates, and wiki grammar notes in `Docs/wiki/anim.md` and `Docs/wiki/material.mgir.md`. Added `FIRGrammarHarmonization_AGIRColonFieldsAndMnemonicIds` and `FIRGrammarHarmonization_MGIRNameTokensAndQualifiedClasses` coverage for the separator/id and name-token/qualified-class counterfactuals.
- `#4-verify-harmonized-tests` `DONE` tester — Verified: ran `system.run_tests` with exact tests `EditorAutomationRpcGateway.core.ir_grammar.AGIRColonFieldsAndMnemonicIds` and `EditorAutomationRpcGateway.core.ir_grammar.MGIRNameTokensAndQualifiedClasses`; job `j_20260515T070930_02dd7d64` resolved both tests, reported no missing tests, and completed with `has_errors: false`.
- `#5-regression` `OPEN` developer — Reopened: axis #3 (class-mnemonic register
  naming) shipped on the assertion *"Recompile is name-blind (the compiler binds
  nodes by graph position, not by token spelling), so the migration is zero-risk
  for round-trip identity"* (lines 56-57, restated in **Fix:** line 145). That
  assertion is FALSE for the AGIR state-machine output pose link, which the
  compiler resolves **by symbol**, not by position: `FAGIRPinResolver::ResolveReference`
  (`AGIRPinResolver.cpp:55`) is a pure `Symbols`-map lookup with no positional
  fallback. The new `%state_machine_<n>` mnemonic id introduced by this axis is
  emitted on the `output` line (`AGIRTextEmitter::EmitStateMachine` pre-allocates
  it via `(void)IdFor(MachineNode)`) but never bound: the emitter writes the
  unassigned name-token opener `state_machine <Name> {` with no `%n = `,
  `ParseBlockHeader` leaves `ResultName` empty, and `AGIRCompiler.cpp` `case
  StateMachine` only did `Symbols.Add` when `ResultName` was non-empty — so
  feeding the verbatim decompiler output of any single-top-level-state-machine
  AnimBP back into the compiler fails with `[AGIR_SYMBOL_NOT_FOUND] Pose
  reference '%state_machine_0' did not resolve to a UAnimGraphNode_Base`. The
  pre-existing `FAGIRRoundTripStateMachineTest` masked this because it only loads
  the Lyra mannequin, whose output is cached-pose-fronted (a `%save_cached_pose_*`
  symbol that DOES carry an explicit binding). Tracked and re-fixed under
  `B-agir-state-machine-output-pose-unbound` (compiler now registers the
  name-token machine under the emitter's positional id; new regression test
  `FAGIRRoundTripTopLevelStateMachineOutputTest` exercises the bare topology).
  This ticket returns to OPEN so the round-trip-identity claim is re-verified
  against the bare single-state-machine shape, not only the cached-pose-fronted
  fixture.
- `#6-wontfix-already-resolved` `WONTFIX` developer — Closing: no outstanding
  code work remains in this ticket's scope. All five harmonization axes are
  shipped and verified green (`#3-harmonized-ir-grammar` + `#4-verify-harmonized-tests`),
  re-confirmed in current source: axis #1 colon fields (`AGIRTextEmitter.cpp`
  `FieldSeparator = ": "` + `%s: %s` field/pose lines), axis #2 MGIR name-token
  entry headers (`MGIRDecompiler.cpp` `FIrTextUtils::FormatNameToken(GetPathName())`),
  axis #3 class-mnemonic ids (`AGIRTextEmitter::MakeLocalId` + per-mnemonic
  `LocalMnemonicCounters`), axis #4 centralized MGIR body indent (no literal
  `"    %"` in `MGIRDecompiler.cpp`), axis #5 qualified MGIR class tokens
  (`ExpressionClassName` → `FormatNameToken(GetClass()->GetPathName())`). The
  `#5-regression` reopen had exactly one residual: re-verify axis #3's
  "recompile is name-blind / zero-risk for round-trip identity" claim against
  the bare single-top-level-state-machine shape. That residual is fully owned
  and already remediated by the IN-REVIEW sibling `B-agir-state-machine-output-pose-unbound`
  (#2-register-name-token-state-machine): the compiler now registers the
  name-token machine under the emitter's positional id
  (`AGIRCompiler.cpp:828-836`, `StateMachineOrdinal`) and ships the load-bearing
  bare-topology regression test `FAGIRRoundTripTopLevelStateMachineOutputTest`
  (`Tests/Assets/TestAnimGraphHandlers.cpp:795`), both confirmed present in
  source. `#5-regression` line 191 already explicitly handed that work to the B
  ticket. Re-implementing here would re-churn settled IR-core code for zero new
  payoff. Note for follow-up (out of THIS ticket's scope): the later-landed
  CRIR/NIR/PCGIR emitters re-introduced the axis #1 `%s=%s` / axis #2 quoted
  name-token divergences this ticket warned about — that is a new harmonization
  pass against the newer IRs, deserving its own ticket, not a reopen of this
  already-completed scope.
