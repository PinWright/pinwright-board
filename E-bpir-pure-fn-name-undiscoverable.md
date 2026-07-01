---
id: E-bpir-pure-fn-name-undiscoverable
title: "BPIR docs name no concrete pure utility functions (string concat Concat_StrStr), forcing an engine-header grep to author a pure data node"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, bpir, compile_bpir, pure-impure, node-discovery, string]
encounters: 5
lastSeen: 2026-06-29T02:34:13Z
---

# BPIR docs classify pure-vs-impure but name no concrete pure utility function

Authoring a common pure data-only node inside a `blueprint.compile_bpir` body —
a string concatenation feeding an exec chain — required leaving the documented
surface. The BPIR wiki has no searchable mapping from the *intent* ("append two
strings, pure") to the engine *function name*: `bpir.pure-impure.md` is a
syntax table that classifies `call Func(...)` as pure-or-impure but names no
concrete utility, and `bpir.examples.format-text.md` shows only `Format`-based
interpolation, naming no pure string-concat function.

So to build the pure `call` the goal demanded, the agent ran a wiki Grep
`pattern:"Concat|Append_Str|BuildString|MakeLiteralString"` over the wiki →
**`No matches found`**, then left the wiki and grepped engine source —
`grep -rn "Concat_StrStr" .../Kismet/KismetStringLibrary.h` →
`static ENGINE_API FString Concat_StrStr(const FString& A, const FString& B);`
— and only then used `call Concat_StrStr(...)` in the BPIR body.

## Why it's process friction (clean outcome, but a C++ detour)

The task was otherwise smooth (15 calls, zero retries, the adversarial
idempotency bet did not reproduce — a verified no-op). The single off-wire
detour was this engine-header lookup to learn one pure function name. It is the
friction-taxonomy "workaround: fallback to reading engine source for a simple
intent" — recoverable in two extra discovery steps, but every BPIR body that
needs a pure utility (string concat especially) pays the same engine-source tax,
because the surface a BPIR author reads (`bpir.*` pages) supplies no
intent → function-name map.

## What it should do

Docs/wiki edit (filed E-/`docs`; the overlay edit is a downstream wiki process,
not part of this ticket). Name the page to improve: **`docs/wiki-src/bpir.pure-impure.md`**
(and/or a small new **`docs/wiki-src/bpir.examples.string-build.md`** worked
example). Add a short list of a handful of high-frequency pure functions by
name, each with a one-line BPIR `call` snippet, so a pure string-append can be
authored without grepping headers:

- string concat — `call Concat_StrStr(A: ..., B: ...)` (also `Append_StrStr`),
  `KismetStringLibrary`;
- a couple of common Kismet conversion/string helpers (e.g.
  `Conv_*ToString`) in the same form.

Keep it a trimmed convenience list, not a re-documentation of every Kismet
function — the point is to remove the "wiki Grep → No matches → engine grep"
detour for the most common pure data node.

**Workaround:** grep `KismetStringLibrary.h` for `Concat_StrStr` (what the task
did), or use the `Format`-based interpolation shown in
`bpir.examples.format-text.md` where it fits (interpolation, not raw concat).

This is the same friction *class* as `E-create-node-operator-symbol-discovery`
(IN-REVIEW) — intent → concrete Kismet function name undiscoverable from the
docs an agent reads, forcing an engine-header grep — but a **distinct surface**:
that ticket covers the low-level `blueprint.graph.create_node`/`CallFunction`
path and edits `docs/wiki-src/blueprint.graph.md`, covering only math/compare
operators + PrintString (no string concat), and explicitly defers whole-graph
authoring to `compile_bpir`. Its fix lands on a page a BPIR author never reads,
so it would not close this gap. Cross-reference, do not merge.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 15 calls, outcome clean, zero retries; the upsert no-op bet did not reproduce). To author the goal's pure data-only node (a string-append feeding PrintString + a var Set), the agent needed the concrete pure function name. Wiki Grep `pattern:"Concat|Append_Str|BuildString|MakeLiteralString"` over the wiki → `No matches found`; agent then grepped engine source → `static ENGINE_API FString Concat_StrStr(const FString& A, const FString& B)` (`KismetStringLibrary.h`), then used `call Concat_StrStr(...)`. `bpir.pure-impure.md` classifies pure/impure but names no concrete utility; `bpir.examples.format-text.md` shows only `Format` interpolation. Propose adding a trimmed list of high-frequency pure functions by name (string concat `Concat_StrStr`/`Append_StrStr`, common `Conv_*ToString`) with one-line BPIR `call` snippets on `docs/wiki-src/bpir.pure-impure.md` (and/or a new `bpir.examples.string-build.md`). Low severity — fully recoverable in two extra discovery steps. Same friction class as `E-create-node-operator-symbol-discovery` (IN-REVIEW) but a distinct method (`compile_bpir` vs `graph.create_node`) and distinct wiki page (BPIR docs vs `blueprint.graph.md`); that ticket's cheat-sheet omits string concat and edits a page BPIR authors don't read, so it would not close this gap. Cross-reference, not a dup.
- `#2-recurrence-math-conv-grep` `OPEN` reporter — Recurrence (cross-task aggregation), distinct functions, same gap. A second clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 13 real RPC calls, outcome clean, zero retries/no `python.execute` fallback). To wire the new custom event's data pins (a Greater-than compare + numeric→string conversions feeding the exec chain), the agent left the MCP/wiki and grepped UE engine headers twice — `pattern:"Greater_DoubleDouble|Conv_DoubleToString|Conv_FloatToString|Add_DoubleDouble"` over `C:/UE_5.7/Engine/Source/Runtime/Engine/Classes` → matched `Add_DoubleDouble`,`Greater_DoubleDouble`; then `pattern:"Conv_DoubleToString|Conv_FloatToString|Conv_IntToString|Conv_DoubleToText"` → matched `Conv_DoubleToString`,`Conv_IntToString` — to confirm the exact UFUNCTION names before authoring the `call` instructions. THINK line: "verify a couple of function names from the engine source so my data wiring uses real nodes." Same intent→name gap as `#1` but for math/compare/conversion pure functions (the proposed `Conv_*ToString` list would have covered the conversions; `Greater_DoubleDouble`/`Add_DoubleDouble` argue for a couple of common math/compare entries too). Prophylactic, not error recovery — the first `compile_bpir` succeeded with the grepped names — so still Low severity, but the second independent occurrence of the same engine-header detour reinforces the trimmed pure-function cheat-sheet on `docs/wiki-src/bpir.pure-impure.md`.
- `#3-recurrence-make-mul-vecconv-grep` `OPEN` reporter — Third independent recurrence (cross-task aggregation), new function families, same gap; from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 15 real RPC calls, outcome clean, zero retries / no `python.execute` fallback, idempotent no-op held). The goal's mixed exec+pure body was `$Health` getter → `Multiply_FloatFloat` → `MakeVector` → `Conv_VectorToString` → `PrintString` (four pure/data nodes reachable only across data pins). After reading ~14 `bpir.*`/`blueprint.*` wiki pages the agent still could not validate the exact callable UFUNCTION names from docs, so before its first `compile_bpir` it dropped to one engine-source grep — TOOL Bash `grep -rho -E "Conv_VectorToString|Multiply_FloatFloat|Conv_DoubleToString|Conv_FloatToString|UFUNCTION.*MakeVector|MakeVector..."` (THINK: "verify a couple of function names in the engine source to de-risk the compile") — then RUN 1 compiled first try (nodeCount:6, status UpToDate). Prophylactic de-risking, not failure recovery. New surfaces vs `#1`/`#2`: a **Make node** (`MakeVector`, `KismetMathLibrary`) and a **struct→string conversion** (`Conv_VectorToString`) — argues the trimmed cheat-sheet on `docs/wiki-src/bpir.pure-impure.md` should also name a couple of common `Make<Struct>` constructors and `Conv_<Struct>ToString` helpers, not only string-concat / math / numeric-conv. (Aside, no cost to this agent: decompile normalized `Multiply_FloatFloat`→`Multiply_DoubleDouble` and the make-sugar→`call MakeVector(...)` consistently across both runs — expected UE5 float-is-double / sugar canonicalization, tracked separately under `E-decompile-bpir-text-not-verbatim-roundtrip`.) Still Low severity — fully recoverable in one extra grep — but the third independent engine-header detour for the same intent→name gap reinforces the proposed pure/utility-function cheat-sheet.
- `#4-recurrence-clamp-mul-isvalid-and-event-spellings` `OPEN` reporter — Fourth independent recurrence (cross-task aggregation), same gap, plus a new sub-angle for **event-name spellings**; from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPCs all `ok`/non-error, zero retries / no `python.execute`; transcript `agent-a6a469a604352e7e0.jsonl`) on a fresh Actor BP `/Game/BP_BpirIdempotencyStress` (3 events BeginPlay/Tick/RecomputeStats whose bodies hang pure data-pin helpers off data pins: IsValid→branch, Add/Greater→branch, FClamp/Multiply/Conv→calls). Before composing the BPIR the agent grepped UE 5.7 engine headers **4×** ("to avoid a compile failure"): `KismetStringLibrary.h` for `Conv_DoubleToString`; `KismetMathLibrary.h` for `FClamp`/`Add_DoubleDouble`/`Greater_DoubleDouble`/`Multiply_DoubleDouble`; `KismetSystemLibrary.h` for `IsValid`/`PrintString`; and `Actor.h` for `ReceiveTick`. None of these exact UFUNCTION names appears in the `bpir.*` wiki pages the agent had just read (`bpir.md`, `bpir.instructions.md`, `bpir.pure-impure.md`, `blueprint.bpir-gotchas.md`). Reinforces `#1`-`#3` for the pure-helper math/clamp/validity surface (`FClamp` and `Multiply_DoubleDouble` are new to the corroboration set, alongside the already-cited `Add_/Greater_DoubleDouble`/`Conv_DoubleToString`), and adds a **new sub-angle the prior entries didn't cover**: the engine *event* spellings (`ReceiveBeginPlay`/`ReceiveTick`) also required an `Actor.h` grep — so the proposed cheat-sheet on `docs/wiki-src/bpir.pure-impure.md` (or a `bpir.functions`/`bpir.instructions` reference) should additionally name the common actor-event spellings, not only pure utilities. Still Low — prophylactic de-risking, the first `compile_bpir` succeeded with the grepped names — but a fourth independent engine-header detour for the same intent→name gap.
- `#5-recurrence-floatfloat-example-misleads` `OPEN` reporter — Fifth independent recurrence (cross-task aggregation), same engine-header-grep gap, **new sub-angle: the trigger is the BPIR wiki examples' own `_FloatFloat` spelling, not a missing name.** From a struggle-audit friction note on a clean `blueprint.compile_bpir` idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPCs all `ok`/non-error, zero retries / no `python.execute`; transcript `agent-a86b84bc6623b01d6.jsonl`) on a fresh Actor BP `/Game/BP_IdemReupsert` (BeginPlay→branch→`ApplyScore(float,int)` custom event reading/writing `Score`+`HitCount`). Friction note verbatim: *"the wiki example used Add_FloatFloat but UE 5.7 BP floats are doubles, so I grepped the engine KismetMathLibrary.h and used Add_DoubleDouble to avoid a compile failure."* Unlike `#1`–`#4` (docs name **no** concrete function → grep to discover), here the docs **do** name one — but the BPIR worked-examples pervasively use the legacy 32-bit `_FloatFloat` variant where UE 5.7 default BP "float" pins are doubles (canonical `_DoubleDouble`): verified `bpir.instructions.md:17` (`Add_FloatFloat`), `bpir.examples.custom-event-with-params.md:9,11` (`Subtract_FloatFloat`/`LessEqual_FloatFloat`), `bpir.examples.else-if-chain.md:9,13` + `bpir.entry-points.md:206` (`Greater_FloatFloat`), `bpir.examples.function-with-return.md:10` (`Multiply_FloatFloat`). So an agent copying the example distrusts the name and greps to confirm `Add_DoubleDouble` (or fears a compile failure). **Key finding:** the float-is-double convention IS already documented — but only on `docs/wiki-src/blueprint.graph.md:341` ("The `_DoubleDouble` suffix is the float (real) pair on UE 5; `float`-typed pins use the `<Op>_FloatFloat` variant…") — a page BPIR authors don't read; **none of the `bpir.*` example pages carry that note.** Proposed fix (in addition to this ticket's pure-function cheat-sheet): add the float-is-double convention note adjacent to the BPIR examples (`docs/wiki-src/bpir.pure-impure.md` and/or `bpir.instructions.md`) and/or switch the five worked examples above to `_DoubleDouble` (the form BP float pins resolve to — cf. `#3`'s observed `Multiply_FloatFloat`→`Multiply_DoubleDouble` decompile normalization, tracked under `E-decompile-bpir-text-not-verbatim-roundtrip`). Cross-ref `E-create-node-operator-symbol-discovery` (IN-REVIEW) whose `#2` cheat-sheet on `blueprint.graph.md` already notes the `_FloatFloat` variant — but on the graph page, not the BPIR pages. Still Low — prophylactic, the `compile_bpir` succeeded — but a fifth engine-header detour for the same intent→name gap.
