---
id: B-bpir-array-element-pin-not-round-trippable
title: "BPIR decompiler emits ``%loop.`Array Element` `` which the BPIR compiler cannot parse"
status: OPEN
severity: Medium
category: bug
tags: [bpir, round-trip, foreach]
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
