---
id: E-bpir-pure-fn-name-undiscoverable
title: "BPIR pages carry no build_api_index/search_api discovery redirect (and no pure string-concat name), so authors grep engine headers for pure UFunction names"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, bpir, compile_bpir, pure-impure, node-discovery, search_api, string]
encounters: 5
lastSeen: 2026-06-29T02:34:13Z
---

# BPIR pages don't point authors at the search_api discovery chain (and name no pure string-concat)

Authoring a common pure data-only node inside a `blueprint.compile_bpir` body —
a string concatenation, a numeric→string conversion, a clamp — repeatedly sent
the agent off the documented surface to grep UE engine headers for the exact
pure UFunction name (e.g. `Concat_StrStr`, `Conv_DoubleToString`, `FClamp`),
across five independent clean tasks (history #1–#5).

**Correction to the original framing (verified against source).** The BPIR pages
do NOT "name no concrete pure utility function" — that premise is false:
`bpir.instructions.md` §2.1 names `GetActorLocation`, `Add_FloatFloat`,
`GetObjectName`; the register-annotation subsection names `Subtract_DoubleDouble`;
§2.7 names `make<Vector>` / `break<HitResult>`; `Format` is documented at §2.5.
The real, narrow gaps are two:

1. **No discovery redirect on any bpir page.** The plugin already ships the
   designed answer to "intent → concrete Kismet function name": the RPC chain
   `blueprint.build_api_index` (`BlueprintApiIndexHandler.cpp:27`) →
   `blueprint.search_api` (`:234`). It is documented — but only on
   `blueprint.graph.md` (`:11`, `:316`, `:318`, `:341`: "run build_api_index …
   then search_api … rather than reading engine headers"), a page a BPIR author
   never opens. Ripgrep for `search_api|build_api_index` over `bpir*.md` → zero
   hits. So a `compile_bpir` author has no pointer to the tool that resolves any
   pure name, and greps headers instead.

2. **No pure string-concat name, and no float-is-double note, on the bpir pages.**
   Ripgrep for `Concat_StrStr|Append_Str|BuildString|MakeLiteralString` over the
   whole `wiki-src` tree → No matches: a pure string-append example is genuinely
   absent. And the `_DoubleDouble` vs `_FloatFloat` convention (history #5 — the
   worked examples use `_FloatFloat` where UE 5.7 float pins are doubles) is
   documented only on `blueprint.graph.md:341`, not on any bpir page.

## Why it's process friction (clean outcome, but a C++ detour)

All five encounters were clean and prophylactic (the first `compile_bpir`
succeeded every time, zero retries, no `python.execute` fallback) — the only
off-wire step was an engine-header grep to confirm a pure UFunction name.
Recoverable in one-two extra discovery steps, so **Low** severity; but every
BPIR body needing a pure utility pays the same tax because the surface a BPIR
author reads carries no pointer to `search_api`.

## What it should do

Docs/wiki edit (E-/`docs`; the overlay edit is a downstream wiki process). Mirror
the structure the sibling `E-create-node-operator-symbol-discovery` (IN-REVIEW)
adopted for the identical friction class — **redirect = core, cheat-sheet =
trimmed convenience** — but on the BPIR surface:

- **(core) Port the `build_api_index` → `search_api` redirect onto a bpir page**
  (`bpir.pure-impure.md`): "for any pure utility whose exact name you don't know,
  run `build_api_index(classFilter=[…])` then `search_api("<verb>")`, searching
  the plain verb, rather than reading engine headers." This is the reusable
  mechanism that resolves ANY pure name — not a static list that can never be
  complete (see the ever-growing history #1→#5: concat → math/conv → make/struct
  → clamp/validity → event-spellings).
- **(convenience) A trimmed pure-utility table** on the same page naming the few
  highest-frequency calls the history actually hit — the previously-absent string
  concat `Concat_StrStr(A,B)`, `Conv_DoubleToString`/`Conv_IntToString`,
  `Conv_VectorToString`, `FClamp` — each a one-line BPIR `call`, explicitly
  framed as a shortcut with `search_api` as the general answer. **Not** an
  unbounded re-documentation of Kismet.
- **(convenience) The float-is-double convention note** (`_DoubleDouble` vs
  `_FloatFloat`) adjacent to the misleading `_FloatFloat` examples
  (`bpir.pure-impure.md` and a one-line note at `bpir.instructions.md` §2.1),
  addressing history #5.

**Workaround:** run `blueprint.build_api_index(classFilter=["KismetStringLibrary"])`
then `blueprint.search_api("concat")` (the documented path), or grep
`KismetStringLibrary.h` for `Concat_StrStr` (what the tasks did).

Same friction *class* as `E-create-node-operator-symbol-discovery` (IN-REVIEW)
but a **distinct surface**: that ticket edits `blueprint.graph.md` (the
`create_node`/`CallFunction` path), covers only math operators + PrintString (no
string concat), and its redirect lands on a page BPIR authors don't read — so it
does not close this gap. Cross-reference, do not merge, do not defer (different
files; it neither gates nor closes this).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 15 calls, outcome clean, zero retries; the upsert no-op bet did not reproduce). To author the goal's pure data-only node (a string-append feeding PrintString + a var Set), the agent needed the concrete pure function name. Wiki Grep `pattern:"Concat|Append_Str|BuildString|MakeLiteralString"` over the wiki → `No matches found`; agent then grepped engine source → `static ENGINE_API FString Concat_StrStr(const FString& A, const FString& B)` (`KismetStringLibrary.h`), then used `call Concat_StrStr(...)`. `bpir.pure-impure.md` classifies pure/impure but names no concrete utility; `bpir.examples.format-text.md` shows only `Format` interpolation. Propose adding a trimmed list of high-frequency pure functions by name (string concat `Concat_StrStr`/`Append_StrStr`, common `Conv_*ToString`) with one-line BPIR `call` snippets on `docs/wiki-src/bpir.pure-impure.md` (and/or a new `bpir.examples.string-build.md`). Low severity — fully recoverable in two extra discovery steps. Same friction class as `E-create-node-operator-symbol-discovery` (IN-REVIEW) but a distinct method (`compile_bpir` vs `graph.create_node`) and distinct wiki page (BPIR docs vs `blueprint.graph.md`); that ticket's cheat-sheet omits string concat and edits a page BPIR authors don't read, so it would not close this gap. Cross-reference, not a dup.
- `#2-recurrence-math-conv-grep` `OPEN` reporter — Recurrence (cross-task aggregation), distinct functions, same gap. A second clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 13 real RPC calls, outcome clean, zero retries/no `python.execute` fallback). To wire the new custom event's data pins (a Greater-than compare + numeric→string conversions feeding the exec chain), the agent left the MCP/wiki and grepped UE engine headers twice — `pattern:"Greater_DoubleDouble|Conv_DoubleToString|Conv_FloatToString|Add_DoubleDouble"` over `C:/UE_5.7/Engine/Source/Runtime/Engine/Classes` → matched `Add_DoubleDouble`,`Greater_DoubleDouble`; then `pattern:"Conv_DoubleToString|Conv_FloatToString|Conv_IntToString|Conv_DoubleToText"` → matched `Conv_DoubleToString`,`Conv_IntToString` — to confirm the exact UFUNCTION names before authoring the `call` instructions. THINK line: "verify a couple of function names from the engine source so my data wiring uses real nodes." Same intent→name gap as `#1` but for math/compare/conversion pure functions (the proposed `Conv_*ToString` list would have covered the conversions; `Greater_DoubleDouble`/`Add_DoubleDouble` argue for a couple of common math/compare entries too). Prophylactic, not error recovery — the first `compile_bpir` succeeded with the grepped names — so still Low severity, but the second independent occurrence of the same engine-header detour reinforces the trimmed pure-function cheat-sheet on `docs/wiki-src/bpir.pure-impure.md`.
- `#3-recurrence-make-mul-vecconv-grep` `OPEN` reporter — Third independent recurrence (cross-task aggregation), new function families, same gap; from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, 15 real RPC calls, outcome clean, zero retries / no `python.execute` fallback, idempotent no-op held). The goal's mixed exec+pure body was `$Health` getter → `Multiply_FloatFloat` → `MakeVector` → `Conv_VectorToString` → `PrintString` (four pure/data nodes reachable only across data pins). After reading ~14 `bpir.*`/`blueprint.*` wiki pages the agent still could not validate the exact callable UFUNCTION names from docs, so before its first `compile_bpir` it dropped to one engine-source grep — TOOL Bash `grep -rho -E "Conv_VectorToString|Multiply_FloatFloat|Conv_DoubleToString|Conv_FloatToString|UFUNCTION.*MakeVector|MakeVector..."` (THINK: "verify a couple of function names in the engine source to de-risk the compile") — then RUN 1 compiled first try (nodeCount:6, status UpToDate). Prophylactic de-risking, not failure recovery. New surfaces vs `#1`/`#2`: a **Make node** (`MakeVector`, `KismetMathLibrary`) and a **struct→string conversion** (`Conv_VectorToString`) — argues the trimmed cheat-sheet on `docs/wiki-src/bpir.pure-impure.md` should also name a couple of common `Make<Struct>` constructors and `Conv_<Struct>ToString` helpers, not only string-concat / math / numeric-conv. (Aside, no cost to this agent: decompile normalized `Multiply_FloatFloat`→`Multiply_DoubleDouble` and the make-sugar→`call MakeVector(...)` consistently across both runs — expected UE5 float-is-double / sugar canonicalization, tracked separately under `E-decompile-bpir-text-not-verbatim-roundtrip`.) Still Low severity — fully recoverable in one extra grep — but the third independent engine-header detour for the same intent→name gap reinforces the proposed pure/utility-function cheat-sheet.
- `#4-recurrence-clamp-mul-isvalid-and-event-spellings` `OPEN` reporter — Fourth independent recurrence (cross-task aggregation), same gap, plus a new sub-angle for **event-name spellings**; from a CallAnalyzer call-trace finding on a clean `blueprint.compile_bpir` idempotency-stress task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPCs all `ok`/non-error, zero retries / no `python.execute`; transcript `agent-a6a469a604352e7e0.jsonl`) on a fresh Actor BP `/Game/BP_BpirIdempotencyStress` (3 events BeginPlay/Tick/RecomputeStats whose bodies hang pure data-pin helpers off data pins: IsValid→branch, Add/Greater→branch, FClamp/Multiply/Conv→calls). Before composing the BPIR the agent grepped UE 5.7 engine headers **4×** ("to avoid a compile failure"): `KismetStringLibrary.h` for `Conv_DoubleToString`; `KismetMathLibrary.h` for `FClamp`/`Add_DoubleDouble`/`Greater_DoubleDouble`/`Multiply_DoubleDouble`; `KismetSystemLibrary.h` for `IsValid`/`PrintString`; and `Actor.h` for `ReceiveTick`. None of these exact UFUNCTION names appears in the `bpir.*` wiki pages the agent had just read (`bpir.md`, `bpir.instructions.md`, `bpir.pure-impure.md`, `blueprint.bpir-gotchas.md`). Reinforces `#1`-`#3` for the pure-helper math/clamp/validity surface (`FClamp` and `Multiply_DoubleDouble` are new to the corroboration set, alongside the already-cited `Add_/Greater_DoubleDouble`/`Conv_DoubleToString`), and adds a **new sub-angle the prior entries didn't cover**: the engine *event* spellings (`ReceiveBeginPlay`/`ReceiveTick`) also required an `Actor.h` grep — so the proposed cheat-sheet on `docs/wiki-src/bpir.pure-impure.md` (or a `bpir.functions`/`bpir.instructions` reference) should additionally name the common actor-event spellings, not only pure utilities. Still Low — prophylactic de-risking, the first `compile_bpir` succeeded with the grepped names — but a fourth independent engine-header detour for the same intent→name gap.
- `#5-recurrence-floatfloat-example-misleads` `OPEN` reporter — Fifth independent recurrence (cross-task aggregation), same engine-header-grep gap, **new sub-angle: the trigger is the BPIR wiki examples' own `_FloatFloat` spelling, not a missing name.** From a struggle-audit friction note on a clean `blueprint.compile_bpir` idempotent re-upsert task (focus `blueprint.compile_bpir`, namespace `blueprint`, outcome `clean`, 18 RPCs all `ok`/non-error, zero retries / no `python.execute`; transcript `agent-a86b84bc6623b01d6.jsonl`) on a fresh Actor BP `/Game/BP_IdemReupsert` (BeginPlay→branch→`ApplyScore(float,int)` custom event reading/writing `Score`+`HitCount`). Friction note verbatim: *"the wiki example used Add_FloatFloat but UE 5.7 BP floats are doubles, so I grepped the engine KismetMathLibrary.h and used Add_DoubleDouble to avoid a compile failure."* Unlike `#1`–`#4` (docs name **no** concrete function → grep to discover), here the docs **do** name one — but the BPIR worked-examples pervasively use the legacy 32-bit `_FloatFloat` variant where UE 5.7 default BP "float" pins are doubles (canonical `_DoubleDouble`): verified `bpir.instructions.md:17` (`Add_FloatFloat`), `bpir.examples.custom-event-with-params.md:9,11` (`Subtract_FloatFloat`/`LessEqual_FloatFloat`), `bpir.examples.else-if-chain.md:9,13` + `bpir.entry-points.md:206` (`Greater_FloatFloat`), `bpir.examples.function-with-return.md:10` (`Multiply_FloatFloat`). So an agent copying the example distrusts the name and greps to confirm `Add_DoubleDouble` (or fears a compile failure). **Key finding:** the float-is-double convention IS already documented — but only on `docs/wiki-src/blueprint.graph.md:341` ("The `_DoubleDouble` suffix is the float (real) pair on UE 5; `float`-typed pins use the `<Op>_FloatFloat` variant…") — a page BPIR authors don't read; **none of the `bpir.*` example pages carry that note.** Proposed fix (in addition to this ticket's pure-function cheat-sheet): add the float-is-double convention note adjacent to the BPIR examples (`docs/wiki-src/bpir.pure-impure.md` and/or `bpir.instructions.md`) and/or switch the five worked examples above to `_DoubleDouble` (the form BP float pins resolve to — cf. `#3`'s observed `Multiply_FloatFloat`→`Multiply_DoubleDouble` decompile normalization, tracked under `E-decompile-bpir-text-not-verbatim-roundtrip`). Cross-ref `E-create-node-operator-symbol-discovery` (IN-REVIEW) whose `#2` cheat-sheet on `blueprint.graph.md` already notes the `_FloatFloat` variant — but on the graph page, not the BPIR pages. Still Low — prophylactic, the `compile_bpir` succeeded — but a fifth engine-header detour for the same intent→name gap.
- `#6-reword-redirect-to-search_api` `IN-REVIEW` developer — Reworded + implemented (all three validity lenses voted reword; verified independently against source). The original title's "names no concrete pure utility function" is FALSE: `bpir.instructions.md:16-18` names GetActorLocation/Add_FloatFloat/GetObjectName, `:58` Subtract_DoubleDouble, `:207-208` break<HitResult>/make<Vector>. The genuine gaps are narrow: (1) no `search_api`/`build_api_index` redirect on any `bpir*.md` (ripgrep → zero hits; the chain exists at `BlueprintApiIndexHandler.cpp:27,234` and is documented only on `blueprint.graph.md:11,316,318,341`), and (2) no pure string-concat name anywhere in wiki-src (`Concat_StrStr|Append_Str|BuildString|MakeLiteralString` → No matches). Rescoped the Fix to mirror the sibling `E-create-node-operator-symbol-discovery` (redirect = core, trimmed cheat-sheet = convenience) rather than an unbounded static list. Implemented: (a) new `## Discovering the concrete pure function name` section on `Docs/wiki-src/bpir.pure-impure.md` — the `build_api_index → search_api` "search the plain verb, not engine headers" redirect, the `_DoubleDouble` vs `_FloatFloat` float-pin convention (addresses #5), and a 4-row convenience table naming Concat_StrStr / Conv_DoubleToString / Conv_IntToString / Conv_VectorToString / FClamp (pin names verified vs UE 5.7 `KismetStringLibrary.h`/`KismetMathLibrary.h`); (b) a one-line float-is-double + search_api note at `Docs/wiki-src/bpir.instructions.md` §2.1, at the misleading `Add_FloatFloat` example. Regression test `Source/PinWright/Private/Tests/Infra/TestBpirPureFnDiscoveryDocs.cpp` (`PinWright.infra.wiki_handler.Topic.BpirPureFnDiscovery`) renders `bpir.pure-impure` through the live `WikiHandler::RenderPage` and asserts the redirect (build_api_index/search_api/"plain verb"/"engine header"), the Concat_StrStr name, and the `_DoubleDouble`/`_FloatFloat` convention — all overlay-exclusive on a topic page, so reverting the section fails the test. Not a dup/defer of the sibling (distinct file `bpir.pure-impure.md` vs `blueprint.graph.md`; it neither gates nor closes this).
