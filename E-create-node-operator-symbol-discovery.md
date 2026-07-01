---
id: E-create-node-operator-symbol-discovery
title: "list_node_types gives no redirect to search_api and no operator-symbol -> KismetMathLibrary target/pin map for CallFunction node-building"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, blueprint-graph, create_node, search_api, node-discovery]
---

# No operator-symbol -> KismetMathLibrary function/pin map; agents fall back to reading engine C++

Building a vanilla math/comparison graph by hand with
`blueprint.graph.create_node nodeType=CallFunction` needs three pieces of
knowledge that the discovery surface an agent naturally reaches for does not
supply:

1. the human operator `-` maps to the UFunction `Subtract_DoubleDouble`,
   `<=` maps to `LessEqual_DoubleDouble` (both in `KismetMathLibrary`), and
   "print" maps to `PrintString` (in `KismetSystemLibrary`);
2. those functions' pin names are `A` / `B` / `ReturnValue` (math) and
   `InString` (PrintString);
3. that the right tool to find all of this is the documented
   `blueprint.build_api_index` -> `blueprint.search_api` chain.

`blueprint.graph.list_node_types` — the catalog an agent hits first for "what
node do I create" — only enumerates K2Node **container** class names
(`VariableGet`, `Branch`, `CallFunction`). It carries no mapping from an
operator symbol or a plain verb ("subtract", "less or equal", "print") to the
underlying `KismetMathLibrary` / `KismetSystemLibrary` UFunction name or its
pins. The two-tier model is documented (`blueprint.graph.md:285-299`: "use
`search_api` for *which function to target* inside a `CallFunction`"), and
BPIR examples even show `Subtract_DoubleDouble` / `LessEqual_FloatFloat`
(`bpir.instructions.md:58`, `bpir.examples.custom-event-with-params.md:11`) —
but none of that is reachable from `list_node_types`, and there is no
operator-symbol cheat-sheet anywhere.

## Why it's process friction (clean outcome, but a C++ detour)

The task ("add a custom event, a `Health - DamageAmount` subtract, a
`Health <= 0` compare, a Branch, a PrintString, by hand node-by-node")
completed cleanly, but only after the agent abandoned the typed-RPC discovery
surface and read engine headers. Friction note verbatim:

> *"`blueprint.graph.list_node_types` only enumerates K2Node CLASS names
> (VariableGet, Branch, CallFunction) and gives no mapping from a human
> operator like '-' or '<=' to the actual KismetMathLibrary function/pin
> names, so I had to read the engine C++ (KismetMathLibrary.h /
> KismetSystemLibrary.h) to discover Subtract_DoubleDouble,
> LessEqual_DoubleDouble, PrintString and their A/B/ReturnValue/InString
> pins; the wiki alone wouldn't have gotten me there."*

The call log corroborates: the agent called `blueprint.graph.list_node_types`
("catalog (paged to disk)") and then went straight to three
`create_node nodeType=CallFunction` calls with already-resolved long-form
names (`Subtract_DoubleDouble`, `LessEqual_DoubleDouble`, `PrintString`).
**`blueprint.search_api` / `blueprint.build_api_index` do not appear anywhere
in the 42-call log** — the documented operator-discovery tool was never used;
the agent read C++ instead. So the gap is two-layered:

- **Discovery-of-the-discovery-tool:** an agent that reaches `list_node_types`
  for `CallFunction` work is not redirected to `search_api`, and concluded
  "the wiki alone wouldn't have gotten me there" despite the chain existing.
- **Symbol -> keyword gap:** even with `search_api`, the agent must already
  know to search the *word* "subtract" / "less equal" rather than the *symbol*
  `-` / `<=`, and that arithmetic/comparison operators are
  `KismetMathLibrary` functions named `<Op>_DoubleDouble` / `<Op>_FloatFloat`.

This is the friction-taxonomy "workaround: fallback to reading engine source
for a simple intent" — recoverable, but every hand-built math/comparison graph
pays the same C++-reading tax.

## What it should do

Docs/wiki edit on `docs/wiki-src/blueprint.graph.md` (this is filed E-/`docs`;
the overlay edit is a downstream wiki process, not part of this ticket). The
**core** fix is the redirect; the cheat-sheet is a trimmed convenience table
that complements BPIR rather than re-documenting its examples.

- **(core) `list_node_types` -> `search_api` redirect.** Right where
  `list_node_types` is described as the node catalog (the `## Cross-cluster
  overlap` block, overlay line 7), add a one-line redirect: "for the *function*
  to target inside a `CallFunction` (math operators, PrintString, etc.) use
  `blueprint.build_api_index` -> `blueprint.search_api`, searching the plain
  verb (`subtract`, `less equal`, `print`) — operators are `KismetMathLibrary`
  `<Op>_DoubleDouble` functions; do not read engine headers." So an agent doing
  a legitimate surgical `create_node`/`CallFunction` fixup (the documented use
  of this low-level RPC) is pointed at the discovery chain instead of C++.
- **(convenience) trimmed operator cheat-sheet.** Add a short `## Operator
  cheat-sheet` table mapping the common symbols to their `KismetMathLibrary` /
  `KismetSystemLibrary` `target` UFunctions and pins (`+`/`-`/`*`/`/` ->
  `<Op>_DoubleDouble` with `A`/`B`->`ReturnValue`, `<=`/`>=`/`==`/`!=` likewise,
  `print` -> `PrintString(InString)`), explicitly noting it is the zero-discovery
  shortcut and that whole-graph authoring should still prefer `compile_bpir`
  (which accepts the same names — see `bpir.examples`). This keeps the table from
  re-documenting BPIR's worked examples while still removing the C++ detour for
  the vanilla hand-build case; the `_FloatFloat` variant is noted in one line.

**Workaround:** read `KismetMathLibrary.h` / `KismetSystemLibrary.h` to learn
the `<Op>_DoubleDouble` / `PrintString` names and their `A`/`B`/`ReturnValue`/
`InString` pins (what the task did), or run
`blueprint.build_api_index(classFilter=["KismetMathLibrary",
"KismetSystemLibrary"])` then `blueprint.search_api("subtract")` /
`search_api("less equal")` / `search_api("print")`.

This is distinct from `E-graph-standard-exec-pin-names` (that covers the fixed
*exec*-pin vocabulary `then`/`execute` only, not function-name or data-pin
discovery) and from the `F-search-api-*` family (those *add* search RPCs for
domains that lack one; BP already has `search_api` — the gap here is that the
operator->function->pin mapping is undiscoverable from where agents look, so
they bypass `search_api` entirely).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean `blueprint.graph` hand-build task (42 calls, outcome clean) constructing BP_DestructibleProp's ApplyDamage -> `Health - DamageAmount` -> `Health <= 0` Branch -> PrintString graph node-by-node. `blueprint.graph.list_node_types` enumerates only K2Node container classes and gives no operator-symbol (`-`, `<=`) -> `KismetMathLibrary` function (`Subtract_DoubleDouble`, `LessEqual_DoubleDouble`) / pin (`A`/`B`/`ReturnValue`, `InString`) mapping, so the agent read engine C++ (`KismetMathLibrary.h`/`KismetSystemLibrary.h`) instead. `blueprint.search_api`/`build_api_index` — the documented operator-discovery chain (`blueprint.graph.md:285-299`) — never appear in the call log: the discovery tool was missed. Propose two `docs/wiki-src/blueprint.graph.md` edits: (1) an operator-symbol cheat-sheet (`+/-/*//`, `<=/>=/==/!=` -> `<Op>_DoubleDouble`, `print` -> `PrintString`) with `A`/`B`/`ReturnValue`/`InString` pins; (2) a redirect from the `list_node_types`/`CallFunction` description to `build_api_index` -> `search_api` searching the plain verb. Low severity — fully recoverable, but every hand-built math/compare graph pays the same engine-source-reading tax. Distinct from `E-graph-standard-exec-pin-names` (exec pins only) and the `F-search-api-*` family (those add missing search RPCs; BP's already exists).
- `#2-redirect-plus-trimmed-cheatsheet` `IN-REVIEW` developer — Rescoped (adversarial lens): made the `list_node_types` -> `search_api` redirect the CORE fix and trimmed the operator cheat-sheet to a convenience table that explicitly defers whole-graph authoring to `compile_bpir`, so it no longer re-documents BPIR's worked examples. Two overlay edits to `Docs/wiki-src/blueprint.graph.md`: (1) in `## Cross-cluster overlap`, after the `list_node_types` line, a redirect telling agents the catalog lists only K2Node container classes and to use `build_api_index` -> `search_api` (searching the plain verb) for the `CallFunction` target instead of engine headers; (2) a new `## Operator cheat-sheet (CallFunction targets)` `##` section — a 9-row table mapping `+`/`-`/`*`/`/`/`<=`/`>=`/`==`/`!=` -> `<Op>_DoubleDouble` (A/B/ReturnValue) and `print` -> `PrintString` (InString), attributed to `KismetMathLibrary`/`KismetSystemLibrary`, with a "prefer compile_bpir for whole graphs" note and the `_FloatFloat` variant called out. Both edits land in `##` sections so they render on the namespace page (where create_node/CallFunction work is read) but stay out of the root index. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestGraphOperatorDiscoveryDocs.cpp` (`EditorAutomationRpcGateway.infra.wiki_handler.Namespace.GraphOperatorDiscovery`) renders the `blueprint.graph` page through the live `WikiHandler::RenderPage` (the HTTP gateway's doc path) and asserts both the cheat-sheet markers (Subtract_DoubleDouble/LessEqual_DoubleDouble/PrintString, ReturnValue/InString, KismetMathLibrary/KismetSystemLibrary) and the redirect markers (build_api_index/search_api/"plain verb") — all overlay-exclusive, so reverting the overlay drops them and the test fails.
