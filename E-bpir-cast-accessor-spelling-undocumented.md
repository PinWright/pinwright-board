---
id: E-bpir-cast-accessor-spelling-undocumented
title: "The cast-result accessor's space-stripped spelling is documented nowhere, and the compiler's own error names the spaced form that does not compile — three plausible spellings failed before the fourth worked"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, cast, pin-name, diagnostic, wiki, discoverability, display-name]
encounters: 2
costly: 1
lastSeen: 2026-09-02T23:10:00+03:00
---

# Reaching a cast result pin whose class name is multi-word is a guessing game

`bpir.instructions` §2.3 documents the accessor with a single-word example:

```
%cast = cast<MyCharacter>(%pawn) [success -> @ok, fail -> @nope]
# Access: %cast.AsMyCharacter
```

Every class whose name UE renders with spaces — every Blueprint Interface (`BPI_…`), and most
`B_`/`BP_`-prefixed Blueprint classes — has a result pin literally named `AsB Drone Game Instance`,
`AsBPI Recoil Receiver`, and so on. Nothing on the page says how to write that, and the three forms
a careful reader would try from the surrounding documentation all fail.

## The four attempts, in the order a reader reaches them

Target: `%c = cast<BPI_RecoilReceiver_C>(%owner) [success -> @have, fail -> @none]`, then calling
the interface function with that pin as `Target:`.

**1. Backticks**, which `bpir.entry-points` § *Backtick-quoted names* presents as the general rule
(*"Any identifier containing spaces must be wrapped in backticks"*) and which
`blueprint.compile_bpir` § *Multi-input-exec target syntax* repeats for pin names:

```
call AddRecoil(Target: %c.`AsBPI Recoil Receiver`, Pitch: $Pitch, Yaw: $Yaw)
-> [COMPILE_FAILED] Line 6: Could not resolve value '%c.`AsBPI Recoil Receiver`' for pin 'Target'
```

**2. `.Result`**, from `bpir.instructions` § *`Array_Get` and accessor aliases*, which states the
aliases resolve **"the sole non-exec output pin"** — a description that fits a cast node exactly (it
has one data output and its other pins are exec):

```
call AddRecoil(Target: %c.Result, …)
-> [COMPILE_FAILED] Line 6: Could not resolve value '%c.Result' for pin 'Target'
```

**3. The name the compiler itself printed.** An earlier failed compile in the same session answered
`TryCreateConnection failed wiring data 'AsBPI Recoil Receiver' -> 'RecoilReceiver'`, so the spaced
form is what the tool says the pin is called. Copying it back is attempt 1, which fails.

**4. Space-stripped, unquoted** — the form that works, arrived at by inference from
`B-bpir-cast-pin-roundtrip-display-name`'s fix note rather than from any wiki page:

```
call AddRecoil(Target: %c.AsBPIRecoilReceiver, Pitch: $Pitch, Yaw: $Yaw)
-> compiled: true
```

## Why this is worth a ticket after `B-bpir-cast-pin-roundtrip-display-name` was fixed

That ticket (DONE) fixed the **decompiler**, so decompiled text now emits the canonical
`AsBDroneGameInstance` and round-trips. It did not document the rule, and it did not touch the two
places a hand-authoring caller actually looks:

- **`bpir.instructions` §2.3's example is single-word**, so it silently implies "class name verbatim"
  and never shows a case where UE's display-name spacing applies.
- **`TryCreateConnection`'s diagnostic still prints the raw pin name**, which for every multi-word
  class is a spelling the compiler will then refuse. A caller who trusts the error message is sent
  to attempt 1.

Neither is a correctness bug; together they cost four compile round-trips on a two-line statement.

## The ask

- One sentence and one example on `bpir.instructions` §2.3: the accessor is `As` + the class name
  with `_C` stripped and **all spaces removed**, never backtick-quoted, e.g.
  `cast<BPI_RecoilReceiver_C>` → `%c.AsBPIRecoilReceiver`. Say explicitly that backticks do **not**
  apply here even though they apply to labels, exec pin names, and function names — that carve-out
  is the whole trap.
- Say on the `Array_Get` accessor-alias table that `.Item`/`.Result`/`.Value` are scoped to
  array-accessor nodes, not to any node with one data output. "The sole non-exec output pin" reads
  as general and is not.
- When `%ref.Pin` fails to resolve, list the candidate accessors the way the multi-input-exec
  diagnostic already lists `available: Enter, Reset`. That single change makes all three dead ends
  self-correcting.

## Dedup

`B-bpir-cast-pin-roundtrip-display-name` (DONE) is the decompiler-side defect this is the
documentation residue of, and its `#4-verify-fix` confirms the canonical form without publishing it.
`B-bpir-dynamic-cast-unknown-diagnostic` is about an unresolvable cast target class, not its result
pin. `B-bpir-func-with-space-unresolvable` is the same family one level up — a spaced *function*
name — and is where the space-stripping resolver fallback came from; note the asymmetry this ticket
is about: function-name resolution has a documented space-stripped fallback **and** accepts
backticks, while pin-reference resolution accepts only the stripped form and rejects backticks. No
ticket covers documenting the cast accessor.

## Severity

**Low**, per the rubric's *"pure friction: docs, discoverability"*. Nothing is unreachable, nothing
returned a false success, and the working form compiles cleanly once known. Reach modifier declined
rather than applied: casts are near-universal in BPIR, which argues up, but only multi-word class
names are affected and single-word ones follow the documented example correctly, which argues back
down. It stays Low with the note that the fix is three sentences and a diagnostic change.

## History
- `#1-filed` `OPEN` reporter — Reaching a cast node's result pin cost four compiles because the accessor spelling for a multi-word class name is documented nowhere and the compiler's own diagnostic prints a form that does not compile. On `%c = cast<BPI_RecoilReceiver_C>(%owner) [success -> @have, fail -> @none]`: (1) the backtick form the wiki mandates for spaced identifiers (`bpir.entry-points` § Backtick-quoted names, echoed by `blueprint.compile_bpir` § Multi-input-exec target syntax) fails — `Could not resolve value '%c.`AsBPI Recoil Receiver`' for pin 'Target'`; (2) `.Result`, documented on `bpir.instructions` § Array_Get accessor aliases as resolving **"the sole non-exec output pin"** — a description a cast node fits exactly — fails the same way; (3) the spaced name printed by an earlier `TryCreateConnection failed wiring data 'AsBPI Recoil Receiver' -> …` in the same session is attempt 1 again; (4) `%c.AsBPIRecoilReceiver`, space-stripped and unquoted, compiles. That form was inferred from `B-bpir-cast-pin-roundtrip-display-name`'s fix note, not from any wiki page: that ticket (DONE) corrected the **decompiler** to emit the canonical spelling and round-trip it, but left `bpir.instructions` §2.3's example single-word — so it silently implies "class name verbatim" and never shows UE's display-name spacing — and left the `TryCreateConnection` diagnostic printing the raw spaced pin name, which for every multi-word class is a spelling the compiler will refuse. Asked for: one sentence plus a multi-word example on §2.3 stating the accessor is `As` + class name with `_C` stripped and all spaces removed and explicitly **not** backtick-quoted despite backticks applying to labels, exec pins and function names; a scope note that `.Item`/`.Result`/`.Value` are array-accessor-only rather than "any node with one data output"; and, when `%ref.Pin` fails, listing the candidate accessors the way the multi-input-exec diagnostic already lists `available: Enter, Reset` — which alone makes all three dead ends self-correcting. Dedup: `B-bpir-cast-pin-roundtrip-display-name` (DONE) is the decompiler defect this is the doc residue of; `B-bpir-dynamic-cast-unknown-diagnostic` is an unresolvable cast target class, not its result pin; `B-bpir-func-with-space-unresolvable` is the same family for function names and is where the space-stripped resolver fallback came from — note the asymmetry, function names accept both the stripped form and backticks while pin references accept only the stripped form. Severity Low (pure friction), reach modifier considered in both directions and declined.
- `#2-ai-stream-bp-class-accessor` `OPEN` reporter — Independent second hit, same session/host (UE 5.8, `EAContentExamples58`), on a Blueprint *class* rather than an interface. Authoring `AIC_Enemy::GetEnemyPawn` I wrote the spelling the wiki's single-word example implies — `%e = cast<BP_EnemyCharacter_C>(%p) [success -> @ok]` then `return %e.AsBP_EnemyCharacter_C` — and got `[COMPILE_FAILED] Line 6: Could not resolve value '%e.AsBP_EnemyCharacter_C' for pin 'ReturnValue'`. What made it worse than `#1`: the two forms behave *differently by class* inside one file. Native-class casts in the same Blueprint took the suffix without complaint — `cast<AIController>` -> `%c.AsAIController` and `cast<BrainComponent>` -> `%bcc.AsBrainComponent` both compiled — so the accessor looked like it worked and only failed on the BP classes, which reads as a mistake in *my* line rather than a spelling rule. The form that worked was the **bare register**, `return %e`, with no accessor at all; `bpir.instructions` §2.3 documents `%cast.AsMyCharacter` as *the* access form and mentions the bare `%ref` only under `Target:` ("pass the base `%ref` as Target"), so nothing says the bare register is also valid in a value position. Suggest the fix in `#1` also state plainly that the bare `%ref` resolves to the cast's primary output everywhere a value is accepted — that single sentence removes the whole guessing game regardless of how the class name is spelled. `encounters` bumped 1 -> 2.
- `#3-rephrase-against-7230b41d` `OPEN` developer — Premise re-checked against current source; two of the four dead ends in `#1` no longer exist. `FBpirValueResolver::ResolveValue` now unwraps a backtick-quoted pin segment (`NormalizeBpirPropertyPath`), so ``%c.`AsBPI Recoil Receiver` `` resolves by exact match, and `FindOutputPinByName`'s accessor-alias fallback applies to any node with exactly one non-exec output, which includes the statement-form cast, so `%c.Result` resolves too. `%c.AsBPIRecoilReceiver` (space-stripped fallback) and the bare `%c` (the cast emitter's `PrimaryOutputPin`) both resolve. The scope note `#1` asked for on the alias table ("array-accessor only") would be false against the code, so it is not made. The real remaining defect: (a) `bpir.instructions` §2.3 still documents the accessor only by a single-word example, so the rule (`As` + the class display name, spaces optional, never the class name verbatim, as `#2` hit with `%e.AsBP_EnemyCharacter_C`) and the bare-register form are written nowhere; (b) an unresolved `%ref.Pin` answers a bare `Could not resolve value '...'` with no candidates; (c) a refused data wire prints the raw spaced pin name (`'AsBPI Recoil Receiver'`), which does not compile as printed.
- `#4-diagnostics-name-compilable-accessor` `IN-REVIEW` developer — `BpirCompiler.cpp`: new `BpirCompilerOutputAccessors` helpers (space-stripped accessor spelling; list of a node's non-exec outputs). In `WireDataPins`, an unresolved `%ref.Pin` whose register has an emitted node now appends `(outputs of %c: AsBPIRecoilReceiver; the bare %c resolves to AsBPIRecoilReceiver)`, and a refused data wire whose value is a `%ref` names its source pin in the space-stripped spelling. `bpir.instructions` §2.3 gains a "Cast result accessor spelling" paragraph with the multi-word example, the backtick form, the verbatim-class-name non-form, the bare register and `.Result`. CHANGELOG entry under 1.0.0. Tests: `PinWright.bpir.compile.CastAccessorSpelling.AcceptedSpellingsCompile` (pins the four documented spellings), `.MisspelledAccessorListsOutputs` and `.RefusedWireNamesCompilableAccessor` (fail if the diagnostic change is reverted), `PinWright.infra.wiki_handler.Topic.BpirCastAccessorSpelling` (doc page). Filter: `PinWright.bpir.compile.CastAccessorSpelling+PinWright.infra.wiki_handler.Topic.BpirCastAccessorSpelling`.
- `#5-result-alias-hidden-pin` `IN-REVIEW` developer — Correction to `#3`: the statement-form cast does NOT have exactly one non-exec output. `UK2Node_DynamicCast::CreateSuccessPin` always creates a `bSuccess` bool output and only hides it on impure casts. The alias fallback counted it, so `%c.Result` still failed, as `#1` attempt 2 reported. Fixed at the root: `FBpirValueResolver::FindOutputPinByName`'s alias fallback now skips `bHidden` pins. The only existing alias user in the tests, `%h.Item` on `Array_Get` in TestCompilerIntegration, has no hidden outputs and is unaffected. The output list in the unresolved-accessor diagnostic also skips hidden pins and stops after 12 names with `, ... (+N more)`. The refused-wire log line uses the space-stripped spelling too. The alias table in `bpir.instructions` now says "sole visible non-exec output pin" and names the cast. `PinWright.bpir.compile.CastAccessorSpelling.AcceptedSpellingsCompile`'s `%typed.Result` case is the failure-direction test for the resolver change. `.MisspelledAccessorListsOutputs` now asserts `outputs of %typed: <canonical>;`, so a listed `bSuccess` fails it.
- `#6-verified-linux` `DONE` tester — Verified on the committed tree, PinWright ae877ccc (pushed to origin/master), UE 5.8 Linux Vulkan. run3/full: offscreen full suite, 5827/5827 passed, 0 failed, 73 skipped. Fix commit 3187fd00. Passed non-skipped in run3/full: `PinWright.bpir.compile.CastAccessorSpelling.AcceptedSpellingsCompile`, `.MisspelledAccessorListsOutputs`, `.RefusedWireNamesCompilableAccessor` and `PinWright.infra.wiki_handler.Topic.BpirCastAccessorSpelling`. Ask 1: `bpir.instructions` §2.3 has a "Cast result accessor spelling" paragraph with the multi-word `BPI_RecoilReceiver_C` example. It says the verbatim class name does not resolve, and it names the space-stripped form, the bare register and `.Result`. On this tree backticks do resolve, so the asked "backticks do not apply" note would now be false. The page documents the backtick form instead, and AcceptedSpellingsCompile compiles all four spellings. Ask 2: the asked "array-accessor only" scope note would also be false. The resolver alias now skips hidden pins, so `%c.Result` reaches the statement cast (the `.Result` case fails without that change, which was `#1` attempt 2). The table says "sole visible non-exec output pin" and names the cast. In both asks the trap is removed in code rather than documented. Ask 3: an unresolved `%ref.Pin` lists `outputs of %c: <compilable spellings>` (capped at 12), and a refused wire names its source pin in the spelling that compiles.
