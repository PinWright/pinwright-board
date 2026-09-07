---
id: E-bpir-cast-accessor-spelling-undocumented
title: "The cast-result accessor's space-stripped spelling is documented nowhere, and the compiler's own error names the spaced form that does not compile — three plausible spellings failed before the fourth worked"
status: OPEN
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
