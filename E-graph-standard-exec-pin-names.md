---
id: E-graph-standard-exec-pin-names
title: "Wiki should publish the fixed standard exec-pin vocabulary so connect_pins skips a discovery round-trip"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, blueprint-graph, connect_pins, pin-name]
---

# Wiki should publish the fixed standard exec-pin vocabulary so `connect_pins` skips a discovery round-trip

The `docs/wiki-src/blueprint.graph.md` overlay repeatedly tells agents to call
`blueprint.graph.get_node_details` (or `_batch`) after creating a node so the
next `blueprint.graph.connect_pins` "doesn't fail on a mismatched pin name"
(overlay lines 7, 279, 283). That guidance is correct for genuinely
variable pin names (cast `success`-vs-`then`, struct members, custom function
parameters), and the overlay's Cast section is a good example of where
discovery is mandatory.

But for the most common wiring case — an event/impure node exec output into
the next impure node's exec input — the pin names are **fixed and universal**:
the K2-standard exec output is `then` (`UEdGraphSchema_K2::PN_Then`; BeginPlay,
CustomEvent, and ordinary CallFunction nodes all expose it), and the standard
exec input on an impure node is `execute` (`PN_Execute`). These never vary by
asset, so calling `get_node_details` purely to relearn them is a pure-overhead
round-trip on every single connect of a vanilla exec chain. The overlay's
blanket "always discover pins first" advice doesn't distinguish the
fixed-vocabulary case from the genuinely-variable one, so a careful agent
discovers even when discovery is unnecessary.

## What it should do

Add a short "standard exec pin names" note to the `blueprint.graph`
overlay (a `##` section, mirroring the existing Operator cheat-sheet, so it
renders on the namespace page alongside the `connect_pins`/`create_node` work)
publishing the fixed vocabulary as a round-trip-skip shortcut:

- exec output (the `then` exec exit on BeginPlay / CustomEvent / impure
  CallFunction) is internally named `then`;
- the standard exec input on an impure node is `execute`;
- Sequence's per-branch exec outputs are `then_0..then_N`;
- **Branch (`K2Node_IfThenElse`) exec outputs are `then` (true) and `else`
  (false)** — NOT `True`/`False`. `True`/`False` are the editor *display*
  labels and the BPIR keywords; the live pin names `connect_pins` resolves
  are `then`/`else` (engine `EdGraphSchema_K2::PN_Then`/`PN_Else`);
- casts remain the documented exception where you must call
  `get_node_details` because the success exec pin may be `then` or `success`
  depending on the cast subclass.

With that published, an agent wiring `BeginPlay.then -> PrintString.execute`
(or `CustomEvent.then -> Delay.execute`) can skip the discovery call entirely
and only fall back to `get_node_details` for the variable-pin cases the
overlay already calls out. Casing is *not* a hard failure — `connect_pins`'
pin lookup is case-insensitive (`FindPinByName` in `BlueprintGraphHelpers.cpp`
falls back to an `ESearchCase::IgnoreCase` compare), so `Then` and `then`
both resolve; the value of publishing the vocabulary is removing the
pre-emptive discovery round-trip, not avoiding a casing error. This is a
docs/wiki edit on `docs/wiki-src/blueprint.graph.md`, not a code change.

Note the Branch row must use the **live** pin names `then`/`else` — these are
what `connect_pins` resolves via `FindPinByName`. `True`/`False` are BPIR-only
keyword aliases the compiler normalizes to `then`/`else`
(`BpirCompiler.cpp:3357-3358`); `connect_pins` does NOT understand them
(`FindPinByName` is case-insensitive but not alias-aware), so publishing
`True`/`False` would induce the very mismatch failure this note prevents.

**Workaround:** call `blueprint.graph.get_nodes` / `get_node_details_batch`
before each `connect_pins` to read the live pin names (what the task did). Note
a wrong guess does not block either: `connect_pins` returns
`availablePins`/`inputPins`/`outputPins`/`closestMatches` in its
`PIN_NOT_FOUND` error (via `BuildPinLookupPayload`), so the connect itself
backstops a miss — the pre-emptive discovery is pure overhead, not a required
recovery step.

**Fix:** add a `## Standard exec-pin vocabulary` section to
`docs/wiki-src/blueprint.graph.md` publishing the fixed names (`then` /
`execute` / `then_0..then_N` / Branch `then`/`else` / cast exception) with the
note that lookup is case-insensitive and the connect error self-describes its
pins. Mirrors the precedent operator cheat-sheet (`E-create-node-operator-symbol-discovery`).

## Evidence

- mcp-fuzz task focused on `blueprint.graph.delete_node` (19 calls, outcome
  clean). Friction note: *"pin exec output is named lowercase 'then' (not
  'Then') and CallFunction exec input is 'execute', so I verified pin names
  via get_nodes/get_node_details_batch before each connect rather than
  guessing."* The call log shows a dedicated `blueprint.graph.get_nodes`
  ("EventGraph pre-wire ids/pins") before the first `connect_pins`, and a
  `blueprint.graph.get_node_details_batch` ("Ping+Delay pin names") before
  the second `connect_pins` — two discovery calls whose only purpose was to
  confirm the fixed `then`/`execute` exec-pin names on standard nodes.
- The overlay never publishes the fixed exec names, so even an agent that
  wants to skip discovery has no documented value to trust and pays the
  round-trip. The reporter framed this as a casing surprise (`then` lowercase,
  not `Then`), but `connect_pins` lookup is case-insensitive
  (`BlueprintGraphHelpers.cpp:113-124`, `ESearchCase::IgnoreCase` fallback), so
  a casing guess does not actually fail — the value of this note is skipping
  the discovery call, not getting casing right. The real, current friction is
  that a careful agent following the overlay's blanket "always discover pins
  first" advice pays an avoidable pre-emptive
  `get_nodes`/`get_node_details_batch` round-trip on every vanilla exec connect.

## History
- `#1-initial-audit` `OPEN` reporter — Standard exec-pin wiring (`BeginPlay.then -> impure.execute`, `CustomEvent.then -> Delay.execute`) forced a `get_nodes` + `get_node_details_batch` discovery pair purely to relearn the fixed, universal exec-pin names (`then` lowercase, `execute`). The `blueprint.graph.md` overlay's blanket "always discover pins" advice doesn't separate the fixed-vocabulary common case from the genuinely-variable cast/struct/param case, so careful agents pay an avoidable round-trip on every vanilla exec connect. Propose the overlay publish the small fixed exec-pin vocabulary (then / execute / then_N / True/False, casts excepted) so discovery can be skipped for the common case. Evidence: mcp-fuzz `blueprint.graph.delete_node` task, friction note quoted above, two discovery calls in a 19-call log.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: corrected the proposed Branch vocabulary from the BPIR-only aliases `True`/`False` to the **live** exec-output names `then`/`else` (`UEdGraphSchema_K2::PN_Then`/`PN_Else`; the `true`/`false` keywords are normalized by the compiler at `BpirCompiler.cpp:3357-3358`, but `connect_pins`' `FindPinByName` is case-insensitive yet NOT alias-aware at `BlueprintGraphHelpers.cpp:120`, so publishing `True`/`False` would induce the very mismatch this note prevents); and struck the false "case-sensitivity surprise trips a guess-based agent / fails silently" framing (casing is irrelevant — the value is skipping the discovery round-trip). Implemented: added a `## Standard exec pin names` `##` section to `Docs/wiki-src/blueprint.graph.md` publishing then/execute/then_N/then+else (Branch)/casts-excepted as a zero-discovery shortcut, mirroring the Operator cheat-sheet pattern. Regression test `Tests/Infra/TestGraphStandardExecPinNamesDocs.cpp` (`PinWright.infra.wiki_handler.Namespace.StandardExecPinNames`) renders the `blueprint.graph` page through `WikiHandler::RenderPage` and asserts the section + the then/execute/then_N/then+else markers and the cast-exception redirect survive — and asserts the page does NOT contain a Branch `True`/`False` exec-pin claim, so re-introducing the wrong alias fails the test. Files: `Docs/wiki-src/blueprint.graph.md`, `Source/PinWright/Private/Tests/Infra/TestGraphStandardExecPinNamesDocs.cpp`.
- `#3-reword-and-publish-vocab` `IN-REVIEW` developer (parallel host) — Independently reworded then implemented the same disposition. Reword (all three validity lenses): the reporter's proposed value "Branch is `True`/`False`" was wrong — `K2Node_IfThenElse` exec outputs are internally `then`/`else` (`EdGraphSchema_K2::PN_Then`=`"then"`/`PN_Else`=`"else"`; `True`/`False` are display labels + BPIR keywords), so publishing it would cause the exact mis-wiring this ticket prevents; corrected to `then`/`else`. Also dropped the "case-sensitivity failure" framing — `connect_pins` lookup is case-insensitive (`BlueprintGraphHelpers.cpp:120`, `ESearchCase::IgnoreCase` fallback), so casing never hard-fails; reframed the value as removing the avoidable pre-discovery round-trip and noted the `PIN_NOT_FOUND` error self-describes pins (`BuildPinLookupPayload`) as the safety net. Implementation: the same `## Standard exec pin names` overlay section on `docs/wiki-src/blueprint.graph.md` also satisfies this host's markers (then/execute/then_0..then_N/Branch then/else/cast exception, case-insensitive note), modeled on the precedent operator cheat-sheet (`E-create-node-operator-symbol-discovery`). Added a second, independent regression test `Source/PinWright/Private/Tests/Infra/TestGraphExecPinVocabularyDocs.cpp` (`PinWright.infra.wiki_handler.Namespace.GraphExecPinVocabulary`) that renders `blueprint.graph` through the live `WikiHandler::RenderPage` (the HTTP gateway's doc path) and asserts the overlay-exclusive markers — including that Branch is published as `else` and the case-insensitive note is present — so reverting the overlay section drops them and the test fails. Both regression tests (`#2`'s `StandardExecPinNames` and this host's `GraphExecPinVocabulary`) are kept and both pass against the merged overlay section.
- `#4-recurrence-delay-completed-label` `OPEN` reporter — Recurred on `level.structure.add_level_blueprint_node` task (build BeginPlay -> Delay -> PrintString on `ExampleProjectWelcome` level BP, outcome clean/ergo). Same exact friction: the agent fired a `blueprint.graph.get_node_details_batch` on the 3 freshly-created nodes purely to confirm the fixed exec-pin vocabulary before wiring — friction note: *"Delay's 'Completed' output surfaces as pin name 'then' (not 'Completed'), which I confirmed via get_node_details before wiring."* This is the specific surprise the overlay should pre-empt: the Delay node's **display label "Completed" maps to the internal exec-pin name `then`**, so an agent reading the node's UI/title vocabulary guesses `Completed` and would mis-wire without a discovery probe. Beyond the generic `then`/`execute` note, the overlay's exec-pin vocabulary should explicitly call out latent-node display-label-vs-pin-name drift (Delay `Completed` = pin `then`) so the canonical wired name is documented. Reinforces the propose: publish the fixed exec-pin vocabulary on `docs/wiki-src/blueprint.graph.md` `connect_pins`/`create_node` so the latent-node case is covered too.
