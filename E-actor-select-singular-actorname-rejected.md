---
id: E-actor-select-singular-actorname-rejected
title: "actor.select requires actorNames (array) and hard-rejects the singular actorName every other per-actor actor.* verb uses — actor.delete already accepts both"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [actor, select, actorname, actornames, singular-plural, param-arity, docs]
---

# `actor.select` rejects the singular `actorName` that every other per-actor `actor.*` verb takes

`actor.select` is the lone array-only identity slot among the per-actor `actor.*`
verbs. It requires **`actorNames`** (a JSON array) and accepts **no** singular
`actorName`, while the verbs an agent has just been calling on a single actor —
`actor.get`, `actor.duplicate`, `actor.add_tag`, `actor.get_transform`,
`actor.set_transform` — all take the singular scalar **`actorName`**. So when a
caller working with one actor reaches for `actor.select` and reuses the same key
they were just using, they get a hard error:

> `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorNames' (type: array)`

…then have to re-spell `actorName:"X"` as `actorNames:["X"]` and retry. One wasted
`is_error` call, corrected on the next call, zero blocked progress — a pure
guessability/round-trip cost, the same arity-mismatch shape as the param-name
drift tickets (`E-actor-verbs-reject-actorpath-slot`,
`E-asset-path-vs-assetpath-list-drift`, `E-geometry-create-name-vs-actorname`),
but the axis here is singular-scalar-vs-plural-array, not a renamed key.

## The asymmetry is concrete and the convenient shape already exists in-namespace

`actor.delete` in the **same** `actor.*` namespace already accepts *both* forms
gracefully and documents it (`LifecycleHandler.cpp:24-27`):

```cpp
REGISTER_RPC_HANDLER("actor.delete", "actor",
  "... Provide actorName for a single actor or actorNames for a batch (one of the two is required) ...",
  RPC_PARAMS(
    RPC_PARAM_OPT("actorName",  "string", "Display label or name of one actor to delete; ignored if actorNames is also provided."),
    RPC_PARAM_OPT("actorNames", "array",  "Array of actor display labels/names; preferred for batch deletion. ...")
  ))
```

`actor.select`, by contrast, declares only the array form
(`ActorSelectionHandler.cpp:20`):

```cpp
REGISTER_RPC_HANDLER("actor.select", "actor", "Select actors in the level editor by name",
  RPC_PARAMS(
    RPC_PARAM_REQ("actorNames", "array", "Array of actor names to select (empty array clears selection)")
  ))
```

So the very ergonomic that would fix this — accept a singular `actorName` scalar
and wrap it into the one-element batch — is already implemented and proven on the
sibling lifecycle verb. `actor.select`'s handler reads `actorNames` directly via
`Payload->TryGetArrayField(TEXT("actorNames"), ...)` (`ActorSelectionHandler.cpp:31`)
and errors `actorNames array is required` (`:33`) with no singular fallback.
(The same array-only requirement holds for `sequencer.add_actors` /
`sequencer.remove_actors` — `SequenceHandler.cpp:786,976` — but those are
explicitly batch verbs; `actor.select` reads as a general selection verb an agent
will routinely aim at a single actor.)

## Why it matters — the process friction

From the audited corridor-pillar task (focus `actor.duplicate`): mid-flail over
an unrelated locked-level failure (filed separately as
`E-actor-duplicate-locked-level-opaque-error`), the agent tried selecting the
source actor before re-duplicating and called
`actor.select {actorName:"Pillar_Source"}` — reusing the singular key it had used
on `actor.spawn`/`actor.set_transform`/`actor.get`/`actor.duplicate` moments
earlier. That hard-failed:

> `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorNames' (type: array)`

The next call `actor.select {actorNames:["Pillar_Source"]}` succeeded. One wasted
call, self-corrected — but the singular→plural re-spell is a predictable trap
every time an agent crosses from the per-actor verbs into `actor.select` with a
single target in hand. The judge's locked-level ticket notes this in passing as a
side-effect of the duplicate flail; it is not the subject of any ticket and has a
distinct root cause (array-only identity slot), so it is filed here as its own
PROCESS angle.

## What it should do

Make `actor.select` accept the singular `actorName` (string) **in addition to**
`actorNames` (array), wrapping a lone string into a one-element selection —
exactly the dual-accept already shipped on `actor.delete` (`LifecycleHandler.cpp:24-27`).
Concretely: change the `actor.select` spec to two `RPC_PARAM_OPT`s
(`actorName` string / `actorNames` array, one-of required) and, in the handler,
fall back to `Ctx.GetString("actorName")` when the `actorNames` array is absent,
pushing the single value into the targets list before the existing selection
loop. Keep the empty-array "clears selection" semantics. The missing-param error
should then enumerate both accepted forms
(`actorName` *or* `actorNames`) instead of naming only `actorNames`, so a
wrong-arity caller self-corrects from the error text without a retry.

If only a docs steer is wanted instead of the handler change, the
`actor.*`-identity asymmetry should be called out on `docs/wiki-src/actor.md`:
note that `actor.select` (and the `sequencer.*_actors` batch verbs) take a plural
`actorNames` **array**, whereas the per-actor verbs (`get`, `duplicate`, `add_tag`,
transform get/set) take the singular `actorName` scalar — so a single-actor
selection must be passed as `actorNames:["X"]`, not `actorName:"X"`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the
  corridor-pillar blockout task (focus `actor.duplicate`; outcome `ergo`). PROCESS
  friction distinct from the judge's `E-actor-duplicate-locked-level-opaque-error`
  (opaque duplicate error) and from `E-actor-verbs-reject-actorpath-slot`
  (`actorPath`-vs-`actorName` key drift): here the axis is singular-scalar vs
  plural-array arity. The agent called `actor.select {actorName:"Pillar_Source"}`
  reusing the singular key it had just used on
  `actor.spawn`/`set_transform`/`get`/`duplicate`, and got
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'actorNames' (type: array)`;
  the retry `actor.select {actorNames:["Pillar_Source"]}` succeeded — one wasted
  call, self-corrected, zero blocked progress. Root cause confirmed in source:
  `actor.select` declares only the array slot (`ActorSelectionHandler.cpp:20`,
  read at `:31`, errors `:33`) with no singular fallback, while the sibling
  `actor.delete` already accepts BOTH `actorName` and `actorNames`
  (`LifecycleHandler.cpp:24-27`) — so the convenient dual-accept fix is already
  proven in-namespace. Dedup (ripgrep over OPEN+closed board files for
  `actorNames`/`actor.select`/singular/plural; qmd unavailable): no existing
  ticket on `actor.select` param arity — the only other hits mention `actorNames`
  only in passing (`E-actor-duplicate-locked-level-opaque-error #1` as a flail
  side-effect) or are unrelated material/pin scalar-array tickets. Proposed fix:
  accept singular `actorName` + alias-listing missing-param error, mirroring
  `actor.delete`; or a `docs/wiki-src/actor.md` steer on the per-actor-singular
  vs select/sequencer-plural asymmetry.
- `#2-dual-accept-fix` `IN-REVIEW` developer — Made `actor.select` dual-accept the
  singular `actorName` scalar in addition to the `actorNames` array, mirroring the
  shipped `actor.delete` pattern. Changed the spec from one `RPC_PARAM_REQ("actorNames")`
  to two `RPC_PARAM_OPT`s (`actorName` string / `actorNames` array) and updated the
  summary to document "Provide actorName for a single actor or actorNames (an array)".
  Handler now reads `actorNames` as an array and, **only when that key is entirely
  absent**, falls back to the singular `actorName` scalar (one-element selection); a
  present-but-empty `actorNames` array still clears the selection (the singular fallback
  is gated on `bHasNamesArray` so the empty-array semantics are preserved). The
  missing-param error now enumerates both forms (`actorName or actorNames required`)
  so a wrong-arity caller self-corrects from the error text without a retry. File:
  `Private/Handlers/Actor/ActorSelectionHandler.cpp`. Also updated the wiki overlay
  `Docs/wiki-src/actor.md` (the per-actor-`actorName` paragraph the adversarial lens
  flagged as actively misleading) to note `actor.select`/`actor.delete` dual-accept
  both forms and that `sequencer.*_actors` stay array-only. Regression tests added to
  `Private/Tests/World/TestActorHandlers.cpp`: `actor.select.SingularActorName` spawns
  a real PointLight, selects it via the singular `actorName` key, and asserts success
  with `selectedCount==1` (reverting to the array-only spec makes the singular call
  hard-fail, failing the test); `actor.select.EmptyArrayClears` asserts a present-but-
  empty `actorNames` array still succeeds with `selectedCount==0` (guards the empty-
  array "clears selection" semantics against the new fallback firing). Not yet compiled
  or unit-tested — a later phase verifies green.
