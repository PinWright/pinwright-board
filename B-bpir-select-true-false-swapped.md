---
id: B-bpir-select-true-false-swapped
title: "BPIR decompile swaps true/false case values on Select node"
status: DONE
severity: Critical
category: bug
tags: [bpir, decompile, select, asset-dump, correctness]
---

# BPIR decompile swaps true/false case values on Select node

`blueprint.decompile` and the `bpir.txt` asset-dump emit the `true:` and `false:` case-value labels on `select(...)` instructions in **swapped** order relative to the actual Blueprint graph. This silently inverts the meaning of any logic that branches on a bool through a Select node, and makes BPIR-driven analysis produce wrong answers about visibility, gating, and state transitions.

This is a correctness bug for BPIR consumers (humans reading the dump *and* agents acting on it) because the text reads plausibly — the syntax is correct, the node graph is intact, only the case-value pairing is wrong.

**Reproducer:**

1. Inspect `Plugins/App/Content/App/UI/LobbyAndMenu/HUD/W_HUD_RaceTrackEnd.uasset` in the Blueprint editor. In `RefreshAnalyzerButtonVisibility`, the Select node has:
   - `False → Collapsed`
   - `True → Visible`
   - Index ← `AND(GetMaxTime>0, IsStandalone)`
   - Return Value → `SetVisibility.InVisibility` on `AnalyzeButton`
2. Read the BPIR dump at `.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/HUD/W_HUD_RaceTrackEnd/bpir.txt:154`:
   ```
   %n5 = select(Index: %n4, true: ESlateVisibility::Collapsed, false: ESlateVisibility::Visible) @(1299, 150)
   ```
3. The text says `true: Collapsed, false: Visible`. The graph says `True: Visible, False: Collapsed`. Swapped.

Net effect: any reader of the BPIR text who reasons "AND=true → button is Collapsed" reaches the opposite conclusion of what the graph actually does. In the case that triggered this report, that misreading led to a confident but wrong claim that the game logic was inverted, when in fact only the BPIR text was inverted.

**Workaround:** Open the asset in the editor and read the Select node's pin labels directly. Do not trust the BPIR text for Select-node case values until this is fixed.

**Fix sketch:**

- Inspect the Select-node decompile path (likely in the BPIR emitter under `Source/PinWright/Private/Decompiler/` — search for `select` instruction emission / `KCST_Select` / `UK2Node_Select`).
- The Select node in UE stores its case pins in `OptionPinNames` / `OptionPinValues`; the emitter probably reads them in the wrong order, or pairs the `true`/`false` literal labels with the wrong option pin.
- Round-trip test: take a known fixture asset whose Select node has distinct values for the true and false branches (e.g. a string-returning Select with `true: "A", false: "B"`), decompile via `blueprint.decompile`, recompile via `blueprint.compile_bpir`, then confirm the output graph still has `true: "A", false: "B"` and not `true: "B", false: "A"`.
- The matching test is probably missing from `bpir-test-matrix.md` — add a Select round-trip case as part of the fix.

**Related cleanup:** If the *compile* path (BPIR text → graph) also reads `true:` and `false:` in the swapped order, the bug may cancel itself out on round-trip and only manifest when a human or LLM reads the intermediate text. Worth checking both directions, not just decompile.

## History

- `#1-initial-repro` `OPEN` reporter — During race-end UI investigation, BPIR decompile of `W_HUD_RaceTrackEnd::RefreshAnalyzerButtonVisibility` reported the Select node as `true: Collapsed, false: Visible`. The actual Blueprint graph has `True: Visible, False: Collapsed` (confirmed by user-supplied screenshot of the graph). This caused a wrong root-cause claim downstream — the graph's logic is correct, only the BPIR text decompile is inverted. Reproducible against `.editor-automation/asset-dumps/App/App/UI/LobbyAndMenu/HUD/W_HUD_RaceTrackEnd/bpir.txt:154`.
- `#2-fix-select-mapping` `IN-REVIEW` developer — Investigated scope: bug is Select-only. Branch (UK2Node_IfThenElse) decompile/compile correctly maps PN_Then→true / PN_Else→false (BpirDecompiler.cpp:939-961, BpirCompiler.cpp:2000-2001); no other nodes use a positional bool→Option-N mapping. Root cause: UE's `UK2Node_Select::AllocateDefaultPins` (k2node-select.cpp:240-251) gives Option 0 the friendly name `False` and Option 1 `True` (`Idx == 0 ? CoreTexts.False : CoreTexts.True`). The plugin's decompiler (BpirTextEmitter.cpp:1822-1828) and compiler (BpirCompiler.cpp:5358-5365) both inverted that mapping symmetrically, so round-trip succeeded but the BPIR text and the asset-dump always lied about which branch each value belonged to. Fix swaps both sides: decompiler now emits `false:` for Option 0 and `true:` for Option 1; compiler now wires BPIR `false:` → Option 0 and `true:` → Option 1. Added regression test `FSelectTrueFalseMappingMatchesUETest` in `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirRoundTrip.cpp` that pins distinct literals through the compile path, then asserts the Select node's Option 0 holds the `false:` literal and Option 1 holds the `true:` literal (UE-side check) and that decompile pairs each label with the correct value (text-side check). Counterfactual: reverting either side of the fix causes the matching half of the test to fail because the asserts no longer rely on round-trip symmetry.
- `#3-verify-select-labels` `DONE` tester — Verified: `asset.search` resolved `W_HUD_RaceTrackEnd` to `/App/App/UI/LobbyAndMenu/HUD/W_HUD_RaceTrackEnd.W_HUD_RaceTrackEnd`, and `blueprint.decompile` emitted bool Select cases with UE Option 0 labeled `false:` and Option 1 labeled `true:` (for example `select(Index: %n0.IsSpectator, false: ESlateVisibility::SelfHitTestInvisible, true: ESlateVisibility::Collapsed)`), matching the fixed Select mapping instead of the old swapped BPIR text.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule. The cited directory `Private/Bpir/` **never existed** — it is not in the pre-rename tree either. The Select mapping this ticket is about is emitter-side, `Decompiler/BpirTextEmitter.cpp:2319` (`Cast<UK2Node_Select>`) with the corrected `Option 0 → false:` / `Option 1 → true:` at `:2341`/`:2345`; the compiler mirror is `Compiler/BpirCompiler.cpp:685`/`:689`. History `#2`'s `BpirTextEmitter.cpp:1822-1828` and `BpirCompiler.cpp:5358-5365` are stale by roughly 500 and 4,700 lines respectively and are left verbatim. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
