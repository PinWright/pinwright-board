---
id: E-scatter-layout-maxpoints-clamped-jitter-refused
title: "`spatial.scatter_layout` silently clamps `maxPoints` past its ceiling but refuses `jitter` past its cap, and the docs give no behaviour for the word \"ceiling\" at all — `maxPoints: 0` clamps to 1 and the over-budget error then reports a \"1-point budget\" the caller never set"
status: OPEN
severity: Medium
category: ergonomic
tags: [spatial, scatter_layout, maxpoints, jitter, clamp, refusal, parameter-validation, error-message, misattributed-value, docs, consistency]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Two caps on one verb, two different behaviours, neither documented

`spatial.scatter_layout` bounds both `jitter` and `maxPoints`. It handles them oppositely.

**`jitter` is refused** — `Handlers/Spatial/ScatterLayoutHandler.cpp:263-270`:

```cpp
const double Jitter = Ctx.GetNumber(TEXT("jitter"), ScatterLayoutDefaultJitter);      // :263
if (Jitter < 0.0 || Jitter > ScatterLayoutMaxJitter)                                  // :264
{
    Ctx.SendError(TEXT("INVALID_PARAMS"),
        FString::Printf(TEXT("jitter is a fraction of spacing and must be in [0, %.2f] "
                             "(got %.4f)."), ScatterLayoutMaxJitter, Jitter));         // :267-268
    return true;
}
```

with the cap and its reasoning at `:57-59`:

```cpp
// Past half a step a point crosses into its neighbour's cell and the lattice stops
// guaranteeing anything about spacing, so the fraction is refused rather than clamped.
constexpr double ScatterLayoutMaxJitter = 0.5;                                        // :59
```

**`maxPoints` is clamped, silently** — `:296-299`:

```cpp
const TOptional<int32> MaxPointsOpt =
    Ctx.GetIntFirstOf({TEXT("maxPoints"), TEXT("max_points")});                       // :296-297
const int32 MaxPoints = FMath::Clamp(MaxPointsOpt.Get(ScatterLayoutDefaultMaxPoints),
    1, ScatterLayoutMaxAllowedPoints);                                                // :298-299
```

against `constexpr int32 ScatterLayoutMaxAllowedPoints = 100000;` (`:70`, default 5000 at `:69`).
No error, no warning, no field saying a clamp occurred. A caller asking for 500000 gets 100000; a
caller asking for 0 gets 1.

## The `maxPoints: 0` case is the one that actively misleads

After the clamp, the budget check at `:355-365` reports the clamped number as though it were the
caller's:

```cpp
if (PlannedPointsD > static_cast<double>(MaxPoints))
{
    Ctx.SendError(TEXT("INVALID_PARAMS"),
        FString::Printf(
            TEXT("The requested lattice is %.0f points (%.0f columns x %.0f rows), over the "
                 "%d-point budget. Nothing was generated: ... "
                 "Raise 'spacing' (currently %.1f cm), shrink 'region', or raise 'maxPoints'."),
```

So `maxPoints: 0` produces *"over the **1**-point budget ... or raise 'maxPoints'"*. The caller
never wrote 1, and — reading `0` the way `actor.list`'s `limit` uses it — most likely meant
"unlimited". They are told to raise a parameter they believe they already opened all the way, using
a number that appears nowhere in their request.

**The namespace already knows `0` is a trap on a cap parameter and handles it explicitly elsewhere.**
`Docs/wiki-src/spatial.md:94`, on `spatial.raycast`'s `maxHits`:

> **Unlike `actor.list`'s `limit`, `0` is not "all"** — each layer costs a line trace — and is
> rejected with `INVALID_PARAMS`.

That is the right pattern and `maxPoints` does not follow it.

## What the docs actually say — correcting the report as it reached me

I was told the documentation calls both caps a "ceiling". It does not, and the real state is worse
rather than better:

- `Docs/wiki-src/spatial.md:392` — jitter: *"`0` emits the bare lattice; anything above `0.5` is
  **refused**."* Behaviour stated, and correct.
- `Docs/wiki-src/spatial.md:397` — maxPoints, in full: *"`maxPoints` (number, default `5000`,
  ceiling `100000`; alias `max_points`)."* The word "ceiling" and **no behaviour at all**.

So the two are not described with one word meaning two things; one is described and the other is
not. And "ceiling" carries no fixed meaning in this namespace — at `spatial.md:94` (`maxHits`,
"default `32`, ceiling `256`") the same word accompanies an explicit rejection, while at `:359`
(`maxResults`, `maxPoses`) it again says nothing. A caller has no way to know from the docs which
caps refuse and which clamp.

The generated page `Saved/PinWright/wiki/spatial.scatter_layout.md` carries both the param row from
`ScatterLayoutHandler.cpp:211-214` and the overlay — the param row says *"A region that would exceed
it is REFUSED with the computed count rather than truncated"*, which is about the lattice, not about
the parameter, and is easy to read as covering both.

## This does NOT contradict the shipped contract, and the distinction matters

`F-scatter-layout-verb`'s `#2` lists among its typed refusals:

> an over-budget lattice (refused with the computed count, never truncated)

That sentence is about the **computed lattice** exceeding the budget. It is accurate as written —
verified: the check at `:355-365` refuses with the computed count and generates nothing, and no
truncation path exists. **This ticket is about the budget *parameter* exceeding its own ceiling** —
a different value on a different code path (`:296-299`, before any lattice is computed). Nothing in
`#2` is being questioned or reopened, and a triager reading this as a returned verification would be
misrepresenting work that is correct.

## What it should do

1. **Pick one policy for both caps and say which in the schema.** Refusing both is the more
   defensible default, since a silently ignored parameter is the harder failure to notice; clamping
   both is acceptable only if the response reports it. Either way, both param descriptions must
   state the behaviour, and `spatial.md:392`/`:397` must match.
2. **Reject `maxPoints: 0` explicitly**, with the `maxHits` wording as the model — `0` is not "all"
   here either, and a caller who means "no cap" wants `100000`.
3. **Never let a clamped value appear in an error message as the caller's.** If the clamp is kept,
   the budget error at `:359-364` should name both numbers — what was requested and what was
   applied — or the message should be generated from the caller's raw input.

## Distinct from

- `F-scatter-layout-verb` (DONE) — the feature ticket that shipped the verb; distinguished in full
  above. Its `#3` tester entry is where this finding was first recorded, and it forward-references
  this file.
- `E-scatter-layout-yaw-range-doc` — the other residual from the same pass, on the response side
  rather than the parameter side.
- `B-actor-list-fields-unknown-key-silently-dropped` — the nearest existing shape on the board (an
  input silently ignored rather than refused), on a different verb and a different parameter kind
  (an unknown key versus an out-of-range value). Same family of complaint, no shared code.

## Dedup

Board-wide search across all statuses for `scatter_layout`, `maxPoints`, `jitter` and clamp/ceiling
parameter tickets: `F-scatter-layout-verb` is the only pre-existing file naming any of them, and it
is the feature ticket, not a defect on this behaviour.

## History
- `#1-one-cap-clamps-one-refuses` `OPEN` reporter — Measured on `spatial.scatter_layout`, then re-derived in source. `jitter` above its cap is refused with `INVALID_PARAMS` (`Handlers/Spatial/ScatterLayoutHandler.cpp:263-270`; cap `ScatterLayoutMaxJitter = 0.5` at `:59`, with the comment at `:57-58` stating refusal is deliberate). `maxPoints` above its ceiling is silently clamped — `FMath::Clamp(MaxPointsOpt.Get(5000), 1, ScatterLayoutMaxAllowedPoints)` at `:298-299`, ceiling `100000` at `:70` — with no error, no warning and no clamp field in the response. `maxPoints: 0` therefore becomes 1, and the budget check at `:355-365` then reports "over the 1-point budget ... or raise 'maxPoints'", a number the caller never set, to a caller who most likely wrote `0` meaning unlimited. The namespace already handles that exact trap elsewhere and does it right: `Docs/wiki-src/spatial.md:94` on `maxHits` — "**Unlike `actor.list`'s `limit`, `0` is not 'all'** ... and is rejected with `INVALID_PARAMS`". Correcting the report as it reached me: the docs do not call both caps a "ceiling". `spatial.md:392` states jitter's behaviour correctly ("anything above `0.5` is refused"); `:397` gives maxPoints as "default `5000`, ceiling `100000`" and **no behaviour at all**, and the same word at `:94` and `:359` accompanies a refusal in one case and nothing in the others — so "ceiling" has no fixed meaning in this namespace, which is a slightly different and slightly worse defect than the one reported. Explicit non-contradiction, recorded because getting it wrong would misrepresent correct work: `F-scatter-layout-verb`'s `#2` "an over-budget lattice (refused with the computed count, never truncated)" is about the **computed lattice** and is accurate as written — verified at `:355-365`, which refuses with the count and has no truncation path. This ticket is about the **budget parameter** exceeding its own ceiling, at `:296-299`, before any lattice is computed. Different value, different code path, `#2` untouched. Ask: pick one policy for both caps and state it in the schema and in `spatial.md`; reject `maxPoints: 0` explicitly on the `maxHits` model; and never let a clamped value be reported as the caller's — name both numbers in the budget error, or build the message from the raw input. Severity Medium: impact class is Medium by the rubric's wording — a caller's value is ignored without notice and the error text then misdirects them, so recovery needs a source dive, nothing in the docs describes the clamp. Not High: the call refuses rather than generating a truncated layout, so no wrong data is produced and nothing is written. Reach bump declined both ways: `spatial.scatter_layout` is not an almost-every-session verb, but it is the namespace's only producer of placements and every scatter goes through it, so this is not a rare edge path either.
