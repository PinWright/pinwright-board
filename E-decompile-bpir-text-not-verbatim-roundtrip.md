---
id: E-decompile-bpir-text-not-verbatim-roundtrip
title: "decompile regenerates BPIR text (labels, SSA temp names, branch-arm order, expanded pin defaults) — round-trip is semantic (topology + coordinates), not literal, and that isn't documented"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, bpir, decompile, round-trip, branch, labels, ssa, defaults]
encounters: 10
lastSeen: 2026-09-05T20:25:00Z
---

# decompile regenerates BPIR text — round-trip is semantic (topology + coordinates), not literal text — and the canonicalizations aren't documented

`blueprint.decompile` is a deterministic *re-emitter*: it reconstructs BPIR text from the
compiled graph rather than reproducing what was authored byte-for-byte. Routing (the
pin -> target *mapping*) is preserved exactly — this is NOT the semantic true/false swap
fixed in DONE `B-bpir-select-true-false-swapped` — and authored `@(x, y)` node coordinates
round-trip byte-exact. But the surface text is canonicalized in several ways:

- **Jump / reconvergence labels are regenerated.** Labels are synthesized at emit time from
  a fixed preferred base (`FEntryState::AllocLabel`, `BpirDecompiler.cpp:313-328`), never
  read back from the authored string. A branch's arms always print `@then` / `@else`
  (authored `@yes`/`@no`, `@truepath`/`@falsepath`, `@big`/`@small` all collapse to these);
  a shared two-predecessor reconvergence prints `@merge` (authored `@done` -> `@merge`, the
  reservation from DONE `B-bpir-decompile-shared-tail-absorbed-into-branch`).
- **SSA temp names are renumbered.** Value names are always numeric
  (`FEntryState::AllocValueName` -> `FormatNumericLocalId`, `BpirDecompiler.cpp:308-311`),
  so authored temps like `%gt`/`%b` come back as `%n0`/`%n1`.
- **Branch target lists print in a canonical order.** `FormatExecTargets` sorts the arm
  parts alphabetically (`BpirTextEmitter.cpp:762`), so a branch always prints
  `[false -> @else, true -> @then]` regardless of authored order — print order only; the
  mapping is intact.
- **Omitted optional pin defaults may be expanded.** Non-object-ref optional pins left off
  the authored `call` (e.g. PrintString `bPrintToScreen`/`TextColor`/`Duration`) are spelled
  out with their engine defaults in the readback (object-ref pins at their canonical "none"
  are still omitted — see the existing `#### Optional-pin omission` note).

Net effect: a consumer doing a round-trip-equivalence check by **diffing decompiled text
against authored text** sees spurious mismatches on labels, temp names, branch order, and
expanded defaults, and may mistake cosmetic normalization for a wiring/coordinate defect. The
general principle is stated for composites at `blueprint.md:156` ("BPIR's round-trip principle
is logical equivalence, not visual fidelity") but is scoped there to composite visual grouping
and does not call out these text-level normalizations; `bpir.entry-points.md:158` sets a
round-trip expectation for authored positions without the text caveat.

## Fix

Docs-only. Add ONE general decompile round-trip caveat — do NOT enumerate every facet as a
growing list (that churns and goes stale). The real fix surfaces are:
- the `### blueprint.decompile` section of `Docs/wiki-src/blueprint.md` (there is **no**
  `blueprint.decompile.md` — the decompile doc is an H3 section inside `blueprint.md`), and
- `Docs/wiki-src/bpir.entry-points.md` (§1b Authored Node Positions, which sets the
  round-trip expectation at the "manually moved nodes round-trip" sentence).

State that decompile *regenerates* the text (labels -> `@then`/`@else`/`@merge`, SSA temps
renumbered, branch arms canonical-ordered false-before-true, omitted defaults expanded), that
only graph topology + authored `@(x, y)` coordinates round-trip, and that round-trip checks
must compare topology + coordinates — confirming exec reconvergence with
`blueprint.graph.get_graph_connections {edgeType: exec}` — not literal label/temp/order text.

**Explicitly de-scoped:** the "single-live-predecessor reconvergence inlined / label dropped"
facet from encounters #2/#5/#6/#9 is NOT a stable decompiler canonicalization — it is a
symptom of the merge node having one predecessor because `B-bpir-fallthrough-reconverge-dropped`
(IN-REVIEW, compiler-side) drops the fall-through exec edge. Once that lands, the merge has two
predecessors and decompile renders `@merge` explicitly. Do not document single-pred inline as a
"can't trust the text" facet here.

## Evidence

Authored-position `event`+`custom_event` body in `/Game/BP_BpirAuthoredPos2`
(round-trip-equivalence probe, focus `blueprint.compile_bpir`). Authored
`@done` + `branch(...) [true -> @then, false -> @else]`. `blueprint.decompile` of the
**saved** graph returned the label as `@merge` and the branch list reordered to
`[false -> @else, true -> @then]`. All eight node coordinates round-tripped exactly
(`@(17,23)`, `@(337,41)`, `@(673,59)`, `@(1009,-103)`, `@(1009,187)`, `@(1345,71)`,
`@(19,503)`, `@(355,521)`) and exec topology was intact (both arms reconverge to the
merge node), confirming this is cosmetic normalization, not a wiring break. Process cost:
because the decompiled text does not literally match the authored BPIR, the probe could
not trust the text and had to spend an extra `blueprint.graph.get_graph_connections`
call to confirm the reconvergence topology — a recurring tax on every round-trip checker
until the canonicalization is documented. No friction beyond that in this run (the agent
was not misled), hence Low.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced by an authored-position
  round-trip-equivalence probe (focus `blueprint.compile_bpir`, asset
  `/Game/BP_BpirAuthoredPos2`). `blueprint.decompile` of the saved graph regenerated the
  authored reconvergence label `@done` as `@merge` and canonicalized the branch target
  list from authored `[true -> @then, false -> @else]` to `[false -> @else, true -> @then]`,
  while node coordinates and exec topology round-tripped exactly. Pure cosmetic text
  normalization (NOT the semantic swap of DONE `B-bpir-select-true-false-swapped`), but
  the two normalizations aren't documented, so a text-diff round-trip checker sees spurious
  mismatches and must cross-check topology via `get_graph_connections`. Fix is docs only:
  state in `blueprint.decompile.md` + `bpir.entry-points.md` that labels are regenerated and
  branch target order is canonicalized, so round-trip comparisons use topology + coordinates,
  not literal text.
- `#2-cross-task-single-predecessor-inline` `OPEN` reporter — Cross-task recurrence with an
  ADDITIONAL canonicalization facet to document: a **single-live-predecessor reconvergence
  target is inlined into its predecessor block and its label name is dropped entirely** — a
  stronger normalization than the `@done`→`@merge` label *regeneration* in `#1` (there the
  label survives renamed; here it disappears). From a clean-process BPIR round-trip-equivalence
  probe (focus `blueprint.compile_bpir`, namespace `blueprint`; fresh Actor BP
  `/Game/BP_BpirRoundTrip`, `event BeginPlay` + `custom_event HandleSignal`, 7 RPCs all
  `ok`/non-error). The agent authored shared reconvergence labels `@done`/`@merged` reached by
  `exec -> @done`/`exec -> @merged` from only the `@then` arm (the `@else`/`@source-absent` arm
  carried no exec jump, so the merge node had a single predecessor). `blueprint.decompile`
  emitted `@then:` with `Valid path` and `Reconverged` back-to-back and **no** `@done:` label,
  and `@else:` ending at `Invalid path` with no visible exec jump — i.e. the shared label was
  collapsed inline and its name not preserved. The decompile was FAITHFUL to the (single-
  predecessor) graph, but the authored-vs-decompiled TEXT no longer lined up on the
  reconvergence labels, so the agent could not confirm exec wiring from the decompile text and
  spent an extra `blueprint.graph.get_graph_connections` (edgeType exec, 8 edges) + a
  `get_node_details_batch` detour to verify topology — the same "can't trust the text, must
  cross-check via get_graph_connections" tax as `#1`. Same fix surface: `blueprint.decompile.md`
  (+ `bpir.entry-points.md`) should also note that **linear-chain / single-predecessor
  reconvergence labels may be inlined and their names dropped**, and that exec reconvergence
  must be confirmed with `get_graph_connections (edgeType:exec)` rather than a text diff.
  Distinct from DONE `B-bpir-decompile-shared-tail-absorbed-into-branch` (that was the
  TWO-predecessor mis-naming *corruption*, now fixed); this single-predecessor inline is correct
  behavior and a docs/expectation gap only. Severity unchanged Low. Dedup: matched this OPEN
  ticket on rg `decompile`/`round-trip`/`label`; appended rather than re-filed.
- `#3-additional-branch-order-and-label-rename-together` `OPEN` reporter — Third independent
  recurrence, this time hitting BOTH canonicalizations on the SAME branch in one shot, with a
  fresh verbatim demo. Authored-position round-trip-equivalence probe (focus
  `blueprint.compile_bpir`, fresh Actor BP `/Game/BP_OracleBranchOrder`, replayed by the oracle).
  Authored `%b = branch(true) [true -> @truepath, false -> @falsepath] @(701, -47)` with custom
  arm labels `@truepath`/`@falsepath`; after compile + `asset.save`, `blueprint.decompile`
  (graphName EventGraph) returned `%n0 = branch(true) [false -> @else, true -> @then] @(701, -47)`
  — the target list reordered to false-before-true AND the authored labels regenerated
  (`@truepath`→`@then`, `@falsepath`→`@else`). All six node coordinates round-tripped EXACTLY
  (`@(13,-53)`, `@(137,-53)`, `@(419,-53)`, `@(701,-47)`, `@(983,-181)`, `@(983,91)`) with no drift
  or grid-snapping, and the routing was preserved (`@then` holds the `"true-branch"` PrintString,
  `@else` the `"false-branch"` — NOT the DONE `B-bpir-select-true-false-swapped` semantic swap),
  confirming pure cosmetic normalization. Reach reinforcement: the round-trip-equivalence attempt
  agent that seeded this oracle run flagged the same combined surprise verbatim — *"the decompiler
  printed the branch bracket as [false, true] and auto-renamed labels truepath→then/falsepath→else,
  which looked like a possible reorder until a pin-level get_node_connections confirmed routing was
  identical — a real user could misread this as a wiring bug"* — i.e. the undocumented `[false,true]`
  + label-rename canonicalization keeps costing every round-trip checker an extra
  `get_node_connections`/`get_graph_connections` cross-check. Same docs-only fix surface
  (`blueprint.decompile.md` + `bpir.entry-points.md`); severity unchanged Low.
- `#4-reach-fourth-recurrence-yes-no-labels` `OPEN` reporter — Fourth independent cross-task
  recurrence (reach reinforcement). Authored-position round-trip-equivalence probe (focus
  `blueprint.compile_bpir`, fresh Actor BP `/Game/BP_BpirRoundTrip`). The attempt authored
  a `branch` with custom arm labels `@yes`/`@no`; after compile + `asset.save`,
  `blueprint.decompile` returned the branch target list as `[false -> @else, true -> @then]`
  — BOTH canonicalizations again: order reordered false-before-true AND the authored labels
  regenerated (`@yes`→`@then`, `@no`→`@else`), confirming the rename is label-agnostic (a
  third authored label pair after `#1`'s `@then/@else` and `#3`'s `@truepath/@falsepath`).
  All 8 authored `@(x,y)` coords round-tripped byte-exact (entry `@(0,0)`, pure RandomBool
  `@(304,0)`, branch `@(608,0)`, arms `@(912,-160)`/`@(912,160)`, custom_event `@(0,400)`,
  set `@(304,400)`, PrintString `@(608,400)`) and routing was preserved (cross-checked via
  `get_graph_connections` — both arms intact, nothing rerouted), so pure cosmetic text
  normalization, NOT a wiring/coordinate defect. Attempt agent's verbatim friction: *"the
  decompiler renames branch labels (@yes/@no to @then/@else) and lists the false arm before
  true, though exec routing is identical."* — same "looked like a reorder until a pin-level
  connections cross-check confirmed routing" tax as `#1`–`#3`. No new facet beyond reach;
  same docs-only fix surface; severity unchanged Low. Note: the seed `compile_bpir` produced
  a fully clean all-positioned round-trip with NO implicit-helper leak (the agent dodged
  `B-bpir-positioned-implicit-helper-not-rejected` via a no-arg pure RandomBool condition +
  a type-matched float member set), so this run is clean on the seed; only the decompile
  canonicalization recurs.
- `#5-cross-task-broken-graph-inline-recurrence` `OPEN` reporter — Fifth cross-task
  recurrence (reach reinforcement). Round-trip-equivalence probe (focus
  `blueprint.compile_bpir`, namespace `blueprint`, fresh Actor BP `/Game/BP_RoundTripProbe`):
  authored a manual-placement `custom_event` whose branch true-arm does `exec -> @merge`
  (explicit) and false-arm falls through to the shared `@merge` terminal. The
  `B-bpir-fallthrough-reconverge-dropped` bug dropped the false-arm edge, so `@merge`
  ("Merged terminal") ended with a SINGLE exec predecessor; `blueprint.decompile` then
  inlined that terminal under the `@then` arm and left `@else` ("False arm") dangling with
  no exec jump and no `@merge` label — the same single-predecessor inline + label-drop
  canonicalization as `#2`. Because the decompiled text cannot distinguish a faithful
  reconvergence from a dropped-edge dead-end, the agent could not confirm exec topology
  from the text and spent **4 extra lower-level topology calls**
  (`blueprint.graph.get_graph_connections` ×2 + `blueprint.graph.find_nodes` ×2) to count
  incoming exec edges (4 vs 5) and map node ids to `@(x,y)` — the same "can't trust the
  text, must cross-check via `get_graph_connections`" tax. All authored coordinates
  round-tripped exactly. The CallAnalyzer proposed a stronger remedy (decompile should
  render reconvergence explicitly), but in the single-predecessor case inlining is faithful
  and the two-predecessor explicit-`@merge` rendering already exists (DONE
  `B-bpir-decompile-shared-tail-absorbed-into-branch`); the residual gap is the
  docs/expectation one this ticket tracks. Dedup: matched this OPEN ticket on rg
  `decompile`/`round-trip`/`reconverge`; appended rather than re-filed. Severity unchanged Low.
- `#6-cross-task-fallthrough-inline-ambiguous` `OPEN` reporter — Sixth cross-task
  recurrence (same single-predecessor-inline facet as `#2`/`#5`; struggle-auditor process angle).
  Authored-position round-trip-equivalence probe (focus `blueprint.compile_bpir`, namespace
  `blueprint`, transcript `agent-a4f4f72eb42e0d70c.jsonl`, 16 RPCs all `ok`/non-error), fresh
  Actor BP `/Game/BP_AuthoredPosStress`: a `custom_event` branch fanned true/false to two
  PrintString arms reconverging at a `Finished` tail; the false arm used the documented
  fall-through, so `B-bpir-fallthrough-reconverge-dropped` left `Finished` with a SINGLE exec
  predecessor. `blueprint.decompile` then **inlined `Finished` into the `@then` (true) block and
  rendered the false arm as a dead end** — a readback indistinguishable from a faithful
  reconvergence, so the agent's THINK explicitly could not tell "whether the decompiler dropped it
  or the edge was never created" and had to drop to `blueprint.graph.get_graph_connections` for
  exec-topology ground truth (invoked **5×** across the run, including a whole second throwaway
  control BP `/Game/BP_AuthoredPosCtrl` to isolate the cause). Same "can't trust the decompile
  text, must cross-check via `get_graph_connections`" tax this ticket tracks. The CallAnalyzer
  re-proposed the stronger "decompile should render reconvergence explicitly" remedy, but (as in
  `#5`) the single-predecessor inline here is *faithful* to the already-broken graph — the
  corrective work is the compiler edge-drop (`B-bpir-fallthrough-reconverge-dropped`) plus this
  ticket's docs note that exec reconvergence must be confirmed with
  `get_graph_connections (edgeType:exec)`, not a decompile text diff. All authored `@(x,y)` coords
  round-tripped exactly (positioning faithful). Dedup: matched this OPEN ticket on rg
  `decompile`/`inline`/`reconverge`; appended rather than re-filed. Severity unchanged Low.
- `#7-cross-task-ssa-temp-renumber-new-facet` `OPEN` reporter — Seventh cross-task recurrence, adding a
  THIRD distinct canonicalization facet not yet documented on this ticket: **SSA temporary names are
  renumbered.** Struggle-audit, transcript `agent-a1cc1fa9088bfccf0.jsonl` (19 RPCs all `ok`/non-error,
  focus `blueprint.compile_bpir`, fresh Actor BP `/Game/BP_BpirRoundTrip`). Authored
  `%gt = call Greater_IntInt(...)`, `%b = branch(%gt) [true -> @big, false -> @small]`, labels
  `@big`/`@small`; `blueprint.decompile` (graph EventGraph) of the saved graph returned
  `%n0: bool = call Greater_IntInt(...)`, `%n1 = branch(%n0) [false -> @else, true -> @then]`, labels
  `@then`/`@else` — i.e. ALL THREE normalizations on one branch: (a) **SSA temps renumbered**
  (`%gt`→`%n0`, `%b`→`%n1`) [NEW — `#1`–`#6` only covered label regeneration + branch-order + inline,
  not temp names], (b) labels regenerated (`@big`→`@then`, `@small`→`@else`), (c) branch target list
  reordered (`[true,false]`→`[false,true]`). All six authored coords round-tripped byte-exact
  (`@(137,-293)`, `@(4203,-451)`, `@(4561,-137)`, `@(4937,-293)`, `@(5311,-77)`, `@(4937,113)`) and the
  node set was exactly the 6 authored nodes, so pure cosmetic text normalization — but because the
  decompiled text was no longer diffable against the authored source, the agent fell back to
  `blueprint.graph.get_graph_connections` (×3) as the wiring source of truth. Docs fix surface unchanged
  (`blueprint.decompile.md` + `bpir.entry-points.md`), but extend the note to state that **SSA temporary
  names (`%gt`→`%n0`) are renumbered** in addition to labels and branch-target order, so round-trip
  comparisons rely on topology + coordinates, not literal temp/label names. Dedup: matched this OPEN
  ticket on rg `decompile`/`round-trip`/`label`; appended rather than re-filed. Severity unchanged Low.
- `#8-cross-task-default-arg-expansion-new-facet` `OPEN` reporter — Eighth cross-task recurrence (struggle
  audit, transcript `agent-aa86aa10d7dde3c34.jsonl`, 7 RPCs all `ok`/non-error, focus
  `blueprint.compile_bpir` mode=append), adding a FOURTH distinct undocumented canonicalization facet:
  **omitted default pin-values are EXPANDED in decompile output.** Upserted a hand-positioned
  `custom_event OnPulseCheck(bool bShouldPulse)` (branch true -> "Pulse ON" print, false -> "Pulse OFF"
  print) into `/Game/ExampleContent/Blueprints/Blueprints/BP_EventGraph`'s populated EventGraph; the two
  `call PrintString(InString: ...)` lines authored only `InString`, but `blueprint.decompile` emitted them
  with the engine defaults spelled out (`bPrintToScreen`, `bPrintToLog`, `TextColor`, `Duration`) — a facet
  not in `#1`–`#7` (which covered label regeneration, branch-target order, single-pred inline, SSA temp
  renumber, but never default-value expansion). Same run also re-hit `#1`/`#3`/`#7`'s label-regen
  (authored `@on`/`@off` -> `@then`/`@else`) and branch-order canonicalization (`[true -> @on, false ->
  @off]` -> `[false -> @else, true -> @then]`) on a PrintString-only fan-out, plus a related label-drop
  facet: the author used an empty trailing `@end` label purely as an arm-terminator (`exec -> @end` on the
  true arm to stop fall-through into the next `@off` block) and the decompiler **elided the empty `@end`
  entirely** (it carried no node) — consistent with the single-pred inline/label-drop facet (`#2`/`#5`/`#6`),
  here for an empty sink-label. All four authored coords round-tripped pixel-exact (`(200,384)`,
  `(520,600)`, `(860,350)`, `(860,600)`), exactly 4 nodes, signature unchanged, pre-existing BeginPlay
  undrifted. Reach/facet-completeness note, NOT a friction repro: unlike `#1`–`#7`, the canonicalization
  cost this attempt ZERO extra calls — it judged the round-trip exact via the planned `decompile` +
  `get_node_connections` and was never misled (CallAnalyzer: "exemplary trace, no inefficiencies"). Same
  docs-only fix surface (`blueprint.decompile.md` + `bpir.entry-points.md`); extend the canonicalization
  list to state that **omitted default pin-values are expanded** and **empty trailing/sink labels are
  elided** so round-trip comparisons rely on topology + coordinates, not literal text. Dedup: matched this
  OPEN ticket on rg `decompile`/`canonical`/`round-trip`; appended rather than re-filed. Severity unchanged Low.
- `#9-cross-task-broken-and-fixed-both-facets` `OPEN` reporter — Ninth cross-task recurrence
  (struggle-audit, transcript `agent-a41684d4aee4673b2.jsonl`, focus `blueprint.compile_bpir`,
  fresh Actor BP `/Game/BP_BpirRoundTrip`). Hit BOTH the broken-graph single-predecessor inline
  (as `#5`/`#6`) AND the `@done`→`@merge` label rename (as `#1`/`#3`) in one run. (a) The v1
  attempt used the documented fall-through, so `B-bpir-fallthrough-reconverge-dropped` left the
  shared tail with a SINGLE exec predecessor; `blueprint.decompile` then emitted `@then:` holding
  BOTH "True path" AND "Reconverged" back-to-back (the shared `@done` label dropped/inlined) while
  `@else:` showed only "False path" with no continuation and **no dead-end marker** — textually
  indistinguishable from a faithful fall-through, so the agent flagged it a "structural concern"
  and fell back to `blueprint.graph.get_graph_connections (edgeType:exec)` for exec ground truth.
  (b) After re-authoring with explicit `exec -> @done` on both arms, `blueprint.decompile` renamed
  the authored `@done` to `@merge` (`exec -> @merge` / `@merge:`) and listed the branch arms
  false-before-true — the same canonicalizations as `#1`/`#3`, here confirmed alongside the inline
  facet in one trace. All 8 authored `@(x,y)` coords round-tripped byte-exact; the only
  non-cosmetic divergence was the dropped fall-through edge (the bug ticket). No new facet; reach
  reinforcement of the "can't trust the decompile text, cross-check via `get_graph_connections`"
  tax. Dedup: matched this OPEN ticket on rg `decompile`/`round-trip`/`label`; appended rather than
  re-filed.
- `#10-reword-and-docs-fix` `IN-REVIEW` developer — Reworded (title/body/Fix/scope) and
  implemented the docs-only fix. (1) Corrected the misattributed fix surface: there is **no**
  `blueprint.decompile.md`; the decompile doc is the `### blueprint.decompile` H3 section inside
  `Docs/wiki-src/blueprint.md`. (2) De-scoped from a growing per-facet enumeration to ONE general
  caveat, and dropped the single-live-predecessor inline "facets" (#2/#5/#6/#9) — those are
  symptoms of `B-bpir-fallthrough-reconverge-dropped` (IN-REVIEW, compiler-side fall-through edge
  drop), not a stable decompiler canonicalization. Source-verified the four stable facets: SSA
  temps always numeric (`FEntryState::AllocValueName` -> `FormatNumericLocalId`,
  `BpirDecompiler.cpp:308-311`); labels synthesized from a fixed base, never the authored string
  (`FEntryState::AllocLabel`, `:313-328`); branch arms added true-then-false (`:1315`/`:1319`) but
  printed alphabetically -> false-before-true (`FormatExecTargets` `Parts.Sort()`,
  `BpirTextEmitter.cpp:762`). Docs edited: added a `#### Round-trip is semantic (topology +
  coordinates), not literal text` subsection to the `### blueprint.decompile` section of
  `Docs/wiki-src/blueprint.md` (labels regenerated -> `@then`/`@else`/`@merge`, SSA temps
  renumbered, branch arms canonical-ordered, omitted defaults expanded; compare topology + coords,
  confirm exec wiring with `get_graph_connections {edgeType:exec}`), plus a matching
  "positions + topology, not literal text" caveat in `Docs/wiki-src/bpir.entry-points.md` §1b
  (corrects the fidelity implication at the "manually moved nodes round-trip" sentence). No
  regression test: docs-only prose change, no production-code behavior change to guard. Severity unchanged Low.
- `#N-round-trip-is-not-merely-inexact-it-FAILS-to-compile` `IN-REVIEW` reporter — An encounter that argues this is more than ergonomic. `blueprint.decompile_function` on `BP_FPSCharacter::BindWeapon` emits, for each bound dispatcher, a pair:

```
%n1 = call Create_Event(self: self, event: @OnWeaponFired)
bind_dispatcher OnFired(Target: %n0.AsBPWeaponBase, event: @OnWeaponFired)
```

  Feeding that text straight back to `blueprint.compile_bpir` **fails**, five times over:
`[COMPILE_FAILED] Unresolved function: 'Create_Event'. Searched: ... and 335 more libraries`, plus a second cascade of `Manual BPIR placement error ... this instruction emitted no primary graph node` because the `@(x, y)` on those lines has nothing to attach to. The fix is to delete every `Create_Event` line and keep only `bind_dispatcher`, which creates the delegate internally — recompiled clean at 14 nodes that way.

  So for a dispatcher-binding body the round trip is not "semantic but not literal": the emitted text is **not valid input to the compiler at all**, and the failure names a function the caller never wrote. That is a different cost from renamed temps or reordered branch arms: a caller who decompiles a working function, edits one line and recompiles — the documented iteration loop — gets five errors about a helper the decompiler invented. Suggest either the decompiler stops emitting the `Create_Event` line for `bind_dispatcher` bodies, or the compiler accepts and ignores it. Either would make this body re-compilable; today only hand-editing does.
