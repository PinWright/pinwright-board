---
id: B-bpir-array-element-pin-not-round-trippable
title: "BPIR decompiler emits ``%loop.`Array Element` `` which the BPIR compiler cannot parse"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, round-trip, foreach]
encounters: 2
costly: 1
lastSeen: 2026-09-07T08:22:00Z
---

# BPIR decompiler emits ``%loop.`Array Element` `` which the BPIR compiler cannot parse

`blueprint.decompile_function` prints a ForEach body's element reference with the pin's
*display* name in backticks:

    %n5 = cast<BP_EnemyCharacter_C>(%n4.`Array Element`) [success -> @ok]

Feeding that exact text back to `blueprint.compile_bpir` fails:

    [COMPILE_FAILED] Line 13: Could not resolve value '%it.`Array Element`' for pin ''

Editor log:

    LogBpirValueResolver: Error: ResolveChainFromPin: VariableGet for '`Array Element`' has no output pin
    LogBpirValueResolver: Error: Value reference '%it.`Array Element`' — no output pin matching
                                 '`Array Element`' on node 'For Each Loop'

The compiler accepts the internal pin name instead: `%it.ArrayElement`.

## Why it matters

Decompile-then-edit-then-recompile is the documented workflow for changing an existing
graph (`blueprint.compile_bpir` § Notes: "To learn the IR for an unfamiliar pattern
quickly, decompile an existing graph and adapt the output"). Any function containing a
ForEach breaks that loop: the round trip fails on text the tool itself produced, and the
error message names neither the accepted spelling nor the fact that a display name was
used. `bpir.instructions.md:341` even states "`foreach`'s `%loop.ArrayElement` is
unaffected" — which is true of the compiler and false of the decompiler.

## Fix

Emit `ArrayElement` (the pin's internal name) from the decompiler, or teach
`ResolveChainFromPin` to match a backticked display name against
`UEdGraphPin::PinFriendlyName` before failing. Either alone closes it; the first is the
smaller change and makes decompiler output canonical.

## Workaround

Replace `` `Array Element` `` with `ArrayElement` in decompiled text before recompiling.

## History

- **#2, 2026-09-07, AI stream.** Hit on `/Game/FPS/AI/AIC_Enemy::UpdateSenses`, a 108-node function
  whose perception loop decompiles as:

      %n9: object<Actor> = foreach(%n8.OutActors) [body -> @body, completed -> @done]
      @body:
          call ConsiderTarget(Candidate: %n9.`Array Element`)

  Feeding that text straight back gives
  `Line 20: Could not resolve value '%n9.\`Array Element\`' for pin 'Candidate'`. Passing the loop
  register bare (`Candidate: %n9`) compiles and is the workaround — the element **is** the register,
  so the `.\`Array Element\`` suffix is pure decompiler noise.

  **Severity Medium -> High by reach.** Not a second stream — the AI stream hit it both times. The
  reach is the workflow: `blueprint.compile_bpir` is upsert-shaped, so the normal way to change one
  branch of a large function is decompile, edit, recompile the whole thing. Any function containing
  a ForEach anywhere therefore fails on a line the author never touched, and the failure is at the
  end of a call carrying the entire function body. On a 108-node function that is an expensive
  round trip to lose to a two-word suffix, and the error names a pin rather than saying "the
  decompiler wrote this and I cannot read it back", so the cause is not obvious from the message.

  Cheapest fix is on the compile side: accept and ignore a `.\`Array Element\`` suffix on a foreach
  register. Fixing the decompiler to stop emitting it is cleaner but breaks nothing either way.

- `#3-unquote-percent-ref-pin-segment` `IN-REVIEW` developer — Fixed on the compile side.
  `Source/PinWright/Private/Compiler/BpirValueResolver.cpp` `ResolveValue` split the `%name.PinName`
  reference with a raw `FindChar('.')` and handed the pin segment to `ResolvePercentRefPin` with its
  backticks intact, so `FindOutputPinByName`'s exact and space-normalized passes compared
  `` `Array Element` `` against `Array Element` and failed. It now splits on the first *top-level* dot
  (`FIrTextUtils::FindTopLevelDelimiterPositions`) and runs the pin path through the file's existing
  `NormalizeBpirPropertyPath` / `NormalizeBpirNameToken` helpers — the same unquoting the sibling
  `$target.Property` branch already used. Decompiler emit is unchanged (the backticked form is legal:
  `docs/wiki-src/bpir.instructions.md` §2.8 already documents backticks for spaced pin names); that doc's
  ForEach section now states both spellings compile. Regression tests, both in
  `Source/PinWright/Private/Tests/Bpir/TestBpirForEachQuotedElementPin.cpp`:
  `PinWright.bpir.compiler.integration.ForEachQuotedElementPinAccess` compiles
  ``call PrintString(InString: %loop.`Array Element`)`` and asserts the macro's Array Element output is
  wired — revert the resolver change and the compile fails with "Could not resolve value" because the
  pin name still carries backticks; and `PinWright.bpir.round_trip.ForEachElementPinRecompiles` compiles
  `%loop.ArrayElement`, decompiles (asserting the emitted text spells it `` .`Array Element` ``) and
  recompiles that text into a fresh Blueprint — revert the change and the recompile fails on a line the
  decompiler itself produced. Not touched: `BpirCompiler.cpp` ~6056 pre-resolves a `Target: %ref.Pin`
  suffix with its own exact-match loop that also ignores backticks; for `foreach` it is harmless
  (`PrimaryOutputPin` *is* the Array Element pin, BpirCompiler.cpp ~6402) so no observable defect there.
