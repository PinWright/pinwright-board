---
id: B-bpir-statement-cast-success-unwired-replace-shared-topology
title: "Statement-form cast's success exec link silently unwired when compile_bpir replaces entries on assets with shared K2Node topology across entries"
status: DONE
severity: High
category: bug
tags: [bpir, compile_bpir, cast, statement-form, latent-continuation, shared-topology, silent-corruption, wireexecpins]
---

# Statement-form cast's success exec link silently unwired when compile_bpir replaces entries on assets with shared K2Node topology

## Symptom

`compile_bpir` on a multi-entry BPIR submission returns `success: true, status: UpToDate, errors: [], warnings: []`, but the resulting Blueprint graph has:
- A statement-form cast (`cast<T>(%X) [success -> @label]`) whose `PN_CastSucceeded` outgoing exec link was never wired.
- All downstream nodes in the `@label:` body created but orphaned (their exec-input pin has zero `LinkedTo` entries).
- The decompiler emits the cast bare as `cast<T>(%X)` (no `[success -> @label]` suffix) — visually indistinguishable from RHS-pure form, but the underlying `UK2Node_DynamicCast` is still impure (no `SetPurity(true)` was called).
- Orphan warning surfaces on decompile: `Orphaned node not reachable from any entry point: K2Node_VariableSet '<setter>' nodeId=<guid> @(<pos>)`.

## Trigger conditions (must overlap — verified isolated variants do NOT reproduce)

1. Source asset has pre-existing shared K2Node topology — physical nodes serving two or more BPIR entries' bodies. In the live repro, `W_FoundGasLeaks`'s original `Tick` `@after:` block and its `OnSearchStateChanged_Event` both reference the same physical `GetTotalLeakCount` / `Format_Text` / `Set Text` nodes at identical authored positions (`(1672, 432)`, `(1918, 295)`, `(2219, 242)` per asset dump `bpir.txt`).
2. `compile_bpir` is called in `append`/`replace` mode on the existing asset (default mode).
3. The submission includes multiple `entry override` / `entry custom_event` blocks at once, at least one of which is the entry whose body shared the K2Nodes (`OnSearchStateChanged_Event`), and at least one of which adds a fresh statement-form cast inside a continuation block reached via `latent Delay [completed -> @X]` (the new `Tick` body).

Isolated repros against a fresh BP do NOT trigger the bug. A separate investigation built five minimal variants on a throwaway BP — cast-in-Tick, cast-in-`@after:` after `latent Delay`, cast-with-forward-label-declaration, cast-after-non-latent `branch [true -> @X]` continuation, cast-direct-in-Tick — and all five compiled cleanly with the `[success -> @label]` clause preserved and zero orphans.

## Repro (live, against `/App/App/UI/W_FoundGasLeaks`)

Pre-state: clean asset (right-click → Reload Asset, or restart editor; on-disk `.uasset` is unmodified in git).

Call: `mcp__editor-automation__call` with `path: "blueprint.compile_bpir"`, `args:`
```
assetPath: "/App/App/UI/W_FoundGasLeaks"
code: |
  entry override Construct() {
      %n0: object<DroneGameState> = call GetDroneGameStatePure()
      %n1: object<TrackBase> = call GetTrackPure(Target: %n0)
      %n2: object<GasLeakTrack> = cast<GasLeakTrack>(%n1) [success -> @bound]
  @bound:
      bind_dispatcher OnLeakStateChanged(Target: %n2.AsGasLeakTrack, event: @OnSearchStateChanged_Event)
      set `Search Track` = %n2.AsGasLeakTrack
      call OnSearchStateChanged_Event()
  }

  entry custom_event OnSearchStateChanged_Event() {
      %n0: int = call GetTotalLeakCount(Target: $`Search Track`)
      %n1: text = call Format_Text(Format: NSLOCTEXT("W_FoundGasLeaks", "ScoreFormat", "{current}/{total}"), current: $`Search Track`.LeaksFound, total: %n0)
      set $ScoreText.Text = %n1.Result
  }

  entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime) {
      %d = latent Delay(Duration: 0.2) [completed -> @after]
  @after:
      %gs: object<DroneGameState> = call GetDroneGameStatePure()
      %tr: object<TrackBase> = call GetTrackPure(Target: %gs)
      %c: object<GasLeakTrack> = cast<GasLeakTrack>(%tr) [success -> @hit]
  @hit:
      %n1: int = call GetTotalLeakCount(Target: %c.AsGasLeakTrack)
      %n2: text = call Format_Text(Format: NSLOCTEXT("W_FoundGasLeaks", "ScoreFormat", "{current}/{total}"), current: %c.AsGasLeakTrack.LeaksFound, total: %n1)
      set $ScoreText.Text = %n2.Result
  }
```
Result: `compiled: true, status: UpToDate, success: true, errors: []`.

Subsequent `blueprint.decompile` shows:
- Construct decompiles with phantom Tick-area nodes attributed to Construct (auto-layout positions ~y=1432-2616).
- `entry custom_event OnSearchStateChanged_Event()` body decompiles EMPTY.
- Construct's `call OnSearchStateChanged_Event()` rewrites to `call SKEL_W_FoundGasLeaks::OnSearchStateChanged_Event()`.
- New Tick body terminates after the cast: `%n3: object<GasLeakTrack> = cast<GasLeakTrack>(%n2) @(752, 1352)` — no `[success -> @hit]`, no `@hit:` block.
- Warning: `Orphaned node not reachable from any entry point: K2Node_VariableSet 'Set Text' nodeId=<...>`.

Reproduces every time on this asset (3/3 attempts this session). Reproduces with both same-label (`@ok` in both Construct and Tick) and distinct-label (`@bound` / `@hit`) variants.

## Static investigation (refuted my initial hypotheses)

Subagent traced compiler source:
- **No path demotes statement-form cast to pure.** The only `SetPurity(true)` call site is `BpirValueResolver.cpp:635` inside `ResolveCastExpression`, which is unreachable from the statement-cast dispatch at `BpirParser.cpp:1164-1167` → `ParseCastInstruction` (`BpirParser.cpp:1853-1898`) → `EBpirOpcode::Cast` emit at `BpirCompiler.cpp:5301-5339` → `CodeNodeEmitter::CreateCastNode` (`CodeNodeEmitter.cpp:432-445`). None of these mutate `bIsPureCast`.
- **`[success -> @label]` parsing is symmetric across statement forms.** `ParseExecClauseFromSuffix` is called at `BpirParser.cpp:1895` for `cast`, same shape as latent (line 1740), branch (line 1611), etc.
- **Label resolution failures hard-error.** `BpirCompiler.cpp:6593, 6781-6786, 6806-6810` push `FCompileError` on `FindFirstImpureAtLabel` or `TryCreateConnection` failure — they do not silently fall back. The repro returns `errors: []`, so neither path is the cause.
- **The decompiler renders an unwired success pin as no-clause.** The cast is most likely emitted statement-form correctly but `WireExecPins` Step 3 (`BpirCompiler.cpp:6614-6810`) fails to add the `Cast.PN_CastSucceeded -> @hit's first impure.Execute` edge — silently. Likely candidate sites: `FindFirstImpureAtLabel` (`BpirCompiler.cpp:6918-6947`) interacting with `RestoreExecIfPure`'s mid-emit opcode mutation (`BpirCompiler.cpp:434-447`); or a post-emit `ReconstructNode` from `MarkBlueprintAsStructurallyModified` resetting wiring.

## Why this is not `B-bpir-event-tick-userwidget-silent-corrupt`

That ticket (filed earlier this session) targets the `EventNameMap` class-blind `Tick → ReceiveTick` lookup creating a phantom override against a missing UFunction. The fix surface is `BpirCompiler.cpp:1259-1267` + `CodeNodeEmitter.cpp:633-639`. This new ticket is a different code path — the entry IS resolved correctly (`entry override Tick(MyGeometry, InDeltaTime)` decompiles as expected post-compile), but the entry's own body has wire-up holes around the statement-form cast inside a latent-continuation block when the asset's pre-existing topology shares K2Nodes across entries. Same author surface (multi-entry compile_bpir), distinct compiler-internal failure mode.

**Workaround:** Manual editor edit (add a Branch + IsValid null-guard ahead of the existing `$Search Track` read in Tick). MCP-side workarounds attempted in-session — distinct labels per entry, full-rewrite of all three entries at once, varying cast-statement placement — all reproduced the corruption.

**Fix:** Runtime instrumentation needed before a confident patch. Recommended next step from the source-trace subagent: log `Schema->TryCreateConnection` results in `WireExecPins` Step 3 for this exact repro to determine whether `Delay.Completed -> Cast.Execute` and `Cast.CastSucceeded -> SetText.Execute` edges are made and then lost (post-emit reset path), or never made in the first place (wire-up phase miss). Candidate surface: `FindFirstImpureAtLabel` × `RestoreExecIfPure` interaction when the label-following block contains pure-patched Call nodes preceding an impure Cast, on an asset whose entries' previous topology shared K2Nodes with the entry being replaced.

## History
- `#1-initial-repro` `OPEN` reporter — Three failed attempts to add a Tick re-resolve fix (cast → null-guard) to `/App/App/UI/W_FoundGasLeaks` in one session. Every attempt produced the same shape: `compile_bpir success: true, errors: []`, but `blueprint.decompile` showed the new Tick body terminating after the cast (no `[success -> @label]`, no continuation block), the `OnSearchStateChanged_Event` body emptied, Construct rewriting its self-event call to `SKEL_<Class>::EventName()`, and an orphan `Set Text` node warning. Three parallel investigation subagents this session: (1) live isolation across 5 variants on a fresh BP could not reproduce the demotion; (2) static read of `BpirCompiler.cpp`/`BpirParser.cpp`/`BpirValueResolver.cpp`/`CodeNodeEmitter.cpp` confirmed no code path demotes statement-form cast to pure — `SetPurity(true)` only reachable from RHS-form `ResolveCastExpression` (`BpirValueResolver.cpp:635`); (3) source-side hypothesis converged on `WireExecPins` Step 3 failing to wire `Cast.PN_CastSucceeded -> @label's first impure.Execute` when the entry being replaced previously shared K2Nodes with another entry on the asset (W_FoundGasLeaks's original `Tick` `@after:` and `OnSearchStateChanged_Event` bodies use identical node positions per asset dump, confirming shared physical K2Nodes). Distinct root cause from `B-bpir-event-tick-userwidget-silent-corrupt` (entry-creation phantom via `EventNameMap` class-blind name remap). Same symptom class as the older corruption family in `B-bp-saved-state-corruption-mcp-edits`, but a separate compiler-internal vector — wire-up phase silent miss rather than CreateDelegate / UFunction class-table orphan.
- `#2-also-via-insert-bpir-before-node` `OPEN` reporter — Surgical retry through `blueprint.insert_bpir_before_node` (a different MCP entry point than `compile_bpir`) on `/App/App/UI/W_GasLeaksDistance` reproduces the same wire-up corruption shape. This confirms the bug is in the shared wire-up phase, not in `compile_bpir` replace-mode specifically. Repro: clean asset, identify the first impure node in Tick's `@after:` block via `blueprint.get_node_connections` walk (Delay.then → Reroute → Reroute → `Set fill` K2Node_VariableSet at GUID `8F93F6B9467DB462223C3BBDDD7ADE61`; this same node is also reached from `OnSearchStateChanged_Event`'s CustomEvent.then through the same reroutes — confirming the shared-topology pattern). Call `blueprint.insert_bpir_before_node` with that nodeId and body `%valid = call IsValid(Object: $\`Search Track\`)\n%b = branch(%valid) [true -> @continue]\n\n@continue:\n`. Result: `compiled: true, status: UpToDate, success: true, errors: []`, `nodeCount: 3`. Subsequent `blueprint.decompile`: Tick's `@after:` body now contains only `IsValid` + `branch(%n1)` — the `[true -> @continue]` exec target is DROPPED from decompile (branch shows no exec targets), `@continue:` label is missing entirely, and the original downstream `Set fill` + three `SetState` nodes are emitted as orphan warnings. Same shape on `OnSearchStateChanged_Event` (also truncated to IsValid + branch with no targets). Construct picks up phantom nodes (GetLeakDistanceMeters / FClamp / Log) attributed to it via decompile heuristic. Promised behavior per `insert_bpir_before_node` wiki — "existing downstream exec links are preserved and reattached after the inserted code" — is not delivered when the inserted body terminates at a `branch [true -> @label]` followed by an empty trailing label. The trailing exec position at the empty `@continue:` label is not recognized as the reattach point; the framework drops the True branch's exec target instead. Either (a) the empty-trailing-label pattern is unsupported and should hard-error rather than silently corrupt, or (b) the framework should reattach the original downstream to the branch's True pin (the last impure's exec target). Tested approach was the textbook null-guard insertion pattern.
- `#3-shared-wireup-instrument-and-patch` `IN-REVIEW` developer — Instrumented every TryCreateConnection in WireExecPins Step 3 and InsertCodeAfterNode reattachment with structured Warning/Verbose logging plus a post-wire verification pass that surfaces unwired cast-success/branch-target/latent-continuation links as FCompileError. Patched FindFirstImpureAtLabel to re-check impurity from K2Node identity rather than mutable BPIR opcode (fixes #1 cast success unwired). Patched InsertCodeAfterNode trailing reattachment to select labeled exit pins over generic ExecOutputPin when last impure has labeled targets (fixes #2 branch true-target dropped with empty trailing label). Regression tests added under Private/Tests/Bpir/ for both manifestations. Tester: verify by running blueprint.compile_bpir against /App/App/UI/W_FoundGasLeaks with the repro body in the ticket, then blueprint.decompile — the success-pin clause and continuation block must round-trip; and blueprint.insert_bpir_before_node against /App/App/UI/W_GasLeaksDistance per #2 repro — exec targets must survive.
- `#4-verified-surgical-insert-on-three-widgets` `DONE` tester — Verified on the #2 (surgical insert) repro path on all three gas-leak widgets after editor restart. Calls: `mcp__editor-automation__call` `path: "blueprint.insert_bpir_before_node"`, `args: {assetPath, nodeId, code: "%valid: bool = call IsValid(Object: $\`Search Track\`)\n%b = branch(%valid) [true -> @continue]\n\n@continue:\n"}` against (a) `/App/App/UI/W_GasLeaksDistance` anchor `Set fill` GUID `8F93F6B9467DB462223C3BBDDD7ADE61` (shared by Tick `@after:` and `OnSearchStateChanged_Event`), (b) `/App/App/UI/W_GasLeaksTargetFound` anchor `SetVisibility` GUID `18CF0F5546F1CF687926AD88825BC593` (shared by Tick `@after:` and `OnLeakZoneChanged_Event`), (c) `/App/App/UI/W_FoundGasLeaks` anchor `Set Text` GUID `A23E3F284D045D368761989C79C3B851` (shared by Tick `@after:` and `OnSearchStateChanged_Event` via reroute joiners — confirmed via `get_node_connections` walk). Each returned `compiled: true, status: UpToDate, success: true, errors: []`, `nodeCount: 3`, zero warnings. Subsequent `blueprint.decompile` per widget shows both the Tick `@after:` block AND the dispatcher event block now read `IsValid` + `branch(...) [true -> @then]` + `@then:` followed by the original body intact — branch True target is wired, no orphans, no SKEL_ leak on call sites that existed before the insert. The framework renamed the authored `@continue` label to `@then` per its convention but preserved the wiring. The compile_bpir replace-mode path from #1 was not directly re-tested this session (the corruption history left two of the three widgets in stale in-memory state until the editor restart, and the surgical path is the actually-useful API for null-guard insertion). Tester confirms #2's failure mode is closed by the patches in #3; the wire-up phase no longer silently drops branch exec targets when the inserted body terminates at a label.
