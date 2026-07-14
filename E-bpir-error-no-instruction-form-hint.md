---
id: E-bpir-error-no-instruction-form-hint
title: "BPIR errors don't suggest the correct instruction form (set-var assignment, call_dispatcher)"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, error-hint, set, call-dispatcher, did-you-mean]
---

# BPIR errors don't suggest the correct instruction form (set-var assignment, call_dispatcher)

Two natural first-attempt mistakes produce dead-end errors with no form hint.

1. Assigning to a Blueprint member variable as `Score = %sum` → `Unrecognized instruction: Score = %sum`. The `=`-assignment path in `BpirParser.cpp` (line 1655) only fires when the LHS is a `%`-register; a bare-identifier LHS falls through to top-level keyword dispatch and hits the generic `Unrecognized instruction` at `BpirParser.cpp:1827`. No hint that member-variable assignment is `set Score = %sum`.
2. Broadcasting an event dispatcher as `call OnScoreChanged(...)` → `Unresolved function: 'OnScoreChanged'` plus a ~395-library search dump at `BpirCompiler.cpp:5705`. That error *does* carry a Hint, but it only covers member-function `Target:` and K2Node full-class-name — it never mentions `call_dispatcher`. The resolver chain (`BpirCompiler.cpp` 5661–5694) never checks member multicast delegates before dumping libraries, even though the dispatcher name is resolvable as a member delegate and `call_dispatcher` is a real verb (`EBpirOpcode::CallDispatcher`, `BpirGrammar.cpp:53`).

Precedent for this already exists: the parser emits a targeted "Did you mean 'entry widget_event …'" for the `@Name:Event` shape (`BpirParser.cpp:807-827`), and the compiler emits pin-name "Did you mean 'X'?" hints (`BpirCompiler.cpp:1345`; ticket `E-bpir-createwidget-pin-hint`). These two shapes just aren't covered.

**Fix:** in `BpirParser` recognize an `ident = value` line (bare-identifier LHS) and suggest `set <ident> = …`; in `BpirCompiler`, before emitting the unresolved-function library dump, check the target class's member multicast delegates and, on a name match, suggest `call_dispatcher <Name>(…)`.

## History
- `#1-initial-repro` `OPEN` reporter — Session repro: (1) `blueprint.compile_bpir` with `Score = %sum` → `Line 3: Unrecognized instruction: Score = %sum`; correct form `set Score = %sum` compiled. (2) `call OnScoreChanged(NewScore: %new)` → `Line 4: Unresolved function: 'OnScoreChanged'. Searched: … and 395 more libraries …`; correct form `call_dispatcher OnScoreChanged(NewScore: %new)` compiled. Verified in source: `BpirParser.cpp:1827` (assignment) and `BpirCompiler.cpp:5705` (dispatcher hint omits `call_dispatcher`) emit no targeted suggestion for either shape.
