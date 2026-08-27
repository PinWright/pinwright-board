---
id: B-ground-actors-prefix-captures-foreign-actors
title: "`spatial.ground_actors` selected by name prefix silently moved six actors belonging to another agent, and because they succeeded no `previousTransform` was echoed — the unintended move is unrecoverable"
status: OPEN
severity: High
category: bug
tags: [spatial, ground_actors, placement, shared-state, concurrency, multi-agent, undo, data-loss, silent-mutation]
encounters: 1
lastSeen: 2026-08-27T19:30:00+05:00
---

# A batch seat matched 89 actors when the caller had spawned 83, moved all 89, and reported only counts

Seating my own floor scatter in a shared editor:

```js
spatial.ground_actors {surface:{preset:"landscape"}, prefix:"RUB_",
                       samples:4, seatPercentile:0.2, embedFraction:0.08,
                       detail:"summary"}
// -> requested:89, placed:83, failed:6, moved:89, totalMatches:89
```

I had spawned **83** `RUB_*` actors. The other six — `RUB_RockA`, `RUB_RockB`, `RUB_RockC`,
`RUB_RubA`, `RUB_RubB`, `RUB_RubC` — belong to another agent working in the same editor, and the
prefix `RUB_` caught them. All six were **moved**. All six then failed verification with
`penetrationCm` 445.0, 445.3, 447.0, 474.2, 489.6 and 496.5, one of them `ACTOR_BURIED`.

## Why the move is unrecoverable

`ground_actors`' own contract (from its wiki page):

> `previousTransform` is echoed for every non-success so you can put it back, or set
> `revertOnFailure`.

Both recovery routes are keyed to **failure**:

- `previousTransform` is echoed for non-successes only. An actor that seats *successfully* but was
  never meant to be selected leaves no record of where it was.
- `revertOnFailure` likewise only fires on a failed post-move check.

There is no "these are the actors I am about to move" surface either: at `detail: "summary"` the
response carries counts and nothing else, so the six foreign names never appeared. They only surfaced
on the *next* call, a `verify_grounding` at `detail: "failures"`, by which point the move had
happened. Had the six seated cleanly they would never have shown up at all.

`totalMatches: 89` against a caller who spawned 83 **is** the available signal, and it is honest —
but it is a number the caller has to independently know the expected value of, printed beside four
other counts, and it does not name anything.

## Suggested fix

1. **Echo `previousTransform` for every moved actor, not only for non-successes** — at least at
   `detail: "all"`. It is the only undo this verb has, and "the move succeeded" is orthogonal to "the
   move was intended".
2. **Name the matched set before mutating**, or make the read verb the sanctioned pre-flight. The
   split the conventions ask for already exists here: `spatial.verify_grounding` is the non-mutating
   sibling and takes the same selectors. The fix may be documentation — `ground-placement`'s
   § *Suggested workflow* already opens with `verify_grounding`, but sells it as "read the failures",
   not as "confirm the selector matches only your actors". In a shared editor the second reading is
   the load-bearing one.
3. Consider an `expectedMatches` guard: when supplied and `totalMatches` differs, refuse with a typed
   error naming the extra actors instead of mutating. That converts a silent capture into a
   pre-flight failure, which is the behaviour a multi-agent build needs.

## Impact

Batch placement verbs selected by name pattern are the documented way to work at scale
(`ground-placement` § *Batch, paging and cost*: "Both verbs are batch-only by construction"). In a
shared editor a name prefix is not a safe scope, and this verb has no dry run, no pre-flight naming,
and no undo for a successful move. Six of another agent's actors were relocated here and cannot be
restored from the response. No editor instability.
