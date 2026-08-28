---
id: B-ground-actors-prefix-captures-foreign-actors
title: "`spatial.ground_actors` selected by name prefix silently moved six actors belonging to another agent, and because they succeeded no `previousTransform` was echoed — the unintended move is unrecoverable"
status: DONE
severity: High
category: bug
tags: [spatial, ground_actors, placement, shared-state, concurrency, multi-agent, undo, data-loss, silent-mutation]
encounters: 3
lastSeen: 2026-08-28T10:40:00+05:00
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

## Encounter 2 — the six were recoverable, and the fix that would have prevented it

**The blind restore to `(0, 0, 0)` was correct, and it is now verified three independent ways.**
Restoring guessed the authoring convention rather than knowing it, so it was checked, not trusted:

1. **Convention.** `/Game/Maps/Atlantis` holds **17** plain-`Actor` HISM holders — 11 in
   `Atlantis/Flora`, 6 in `Atlantis/Rubble`. All 17 sit at identity: location `(0,0,0)`, rotation
   zero, scale 1. The 11 flora holders were never touched by the bad call, so they are an
   uncontaminated control for what a holder in this level is supposed to be.
2. **Reproduction of the author's own number.** The rubble agent reported its instances at
   "lowest corner 20–90 cm below ground". Re-measuring that same quantity now — lowest transformed
   AABB corner minus the `LandscapeProxy` height under it, 194 instances sampled across all six
   holders — gives **−20.1 … −98.9 cm, median −32.5**, with **zero** samples above ground. A residual
   holder offset δ would shift every one of those numbers by δ; the shallow edge landing on −20.1
   against an authored floor of −20 pins **|δ| ≲ 1 cm**.
3. **Vision.** Low-angle captures at three widely separated sites — (2200, 2900) looking NE,
   (5659, 6100) looking back SW over the same field, and (−8800, −9200) in the opposite quadrant —
   show rocks and rubble meeting the sand with contact shadows and no daylight beneath any of them.

Measured with `r.ScreenPercentage 100` (this editor defaults to 50) and exposure pinned min=max=1.

### The fix this ticket should carry

The three suggestions above are all about *scoping the selector*. They are worth doing, but none of
them addresses what actually happened, and a caller who scoped perfectly could still hit it:
**seating a HISM holder is meaningless for any selector.** `ground_actors` fits the actor's
world AABB to the ground; for a holder whose component owns instances authored in world space that
AABB is the bounding box of the entire scatter — here 24000 × 24000 — so the "footprint" it samples
is the whole map and the seat lifts every instance together. The verb reported `placed` for six
actors it had moved 445–496 cm, because the post-move re-measurement agreed with a solve that was
itself meaningless.

Add a fourth, and put it first: **refuse an actor whose seated bounds come from an
`UInstancedStaticMeshComponent` / `UHierarchicalInstancedStaticMeshComponent`**, with a typed
`HOLDER_NOT_SEATABLE` naming the component and its instance count, and pointing the caller at
grounding the instance transforms when they are computed. It is a one-predicate check, it needs no
new caller discipline, and unlike the selector fixes it cannot be defeated by a name collision. The
same predicate belongs on `verify_grounding`, which will otherwise keep answering a question about
the scatter's bounding box as though it were about a prop.

`previousTransform` on every moved actor (suggestion 1) is still the one that would have made this
recoverable without inference, and it remains the highest-value change for a shared editor.

## History

- `#1-reported` `OPEN` reporter — original capture of six foreign actors, unrecoverable from the
  response.
- `#2-restore-verified-and-root-fix` `OPEN` reporter — Restore to identity confirmed correct by the
  three checks above; 564 instances intact and bedded. Adds the `HOLDER_NOT_SEATABLE` refusal as the
  root-cause fix, since every selector-scoping remedy leaves the underlying "a HISM holder has no
  seatable footprint" defect in place. Status left `OPEN` — reporter, not fixer.
- `#3-scope-guard-and-undo-record` `IN-REVIEW` developer — Both defects in the ticket title fixed in
  `Source/PinWright/Private/Handlers/Spatial/GroundPlacementHandler.cpp` (`spatial.ground_actors`
  only; `verify_grounding` untouched). (1) Selector scope: a pattern selector (`prefix` / `filter`)
  on the mutating verb now requires a new `expectedMatches` param; without it the call is refused
  `MISSING_REQUIRED_PARAM` naming the count it would have matched, and when the stated count
  disagrees with the level it is refused `MATCH_COUNT_MISMATCH` *before the first move*, carrying
  `matchedActors[]` (capped at 64) so the caller can see which actors are not theirs. `actors` /
  `selection` are exempt (they already enumerate what the caller named) but honour the value when
  given; `verify_grounding` stays free of it and is the sanctioned pre-flight. The guard runs after
  actor resolution, so a typo'd pattern still reads `NO_ACTORS_MATCHED`. (2) Undo: `previousTransform`
  is now echoed for **every** actor with a captured pre-move transform, successes included (was
  gated on `!IsSeated()`), and a new `movedActors[]` array — one `{actor, path, previousTransform}`
  entry per actor actually moved — is emitted at **every** `detail` level including `summary`,
  because it is the receipt for a mutation rather than a diagnostic. Reverted actors are excluded
  (`WasMoved()` is false and they are already back). Regression tests added to
  `Source/PinWright/Private/Tests/Spatial/TestGroundPlacement.cpp`:
  `PinWright.spatial.ground_actors.PatternSelectorNeedsExpectedMatches` and
  `PinWright.spatial.ground_actors.MovedActorsEchoPreviousTransform`. Needs one new error code
  `ERR_MATCH_COUNT_MISMATCH = "MATCH_COUNT_MISMATCH"` in `Handlers/ErrorCodes.h` (owned by the
  orchestrator) or the emit-registration test fails. **Not addressed here:** encounter 2's
  `HOLDER_NOT_SEATABLE` refusal for ISM/HISM holders — it belongs on both verbs, and
  `verify_grounding` is another agent's file scope this wave.
- `#4-verified-fixed` `DONE` verifier — 2026-08-28. Both defects in the title re-tested live on the
  editor rebuilt at `b79ba53e`, on `/Game/Maps/Atlantis`. The verb has no dry run, so the over-match
  was staged on two throwaway cubes (`PWVERIFY_GA_Mine_01`, `PWVERIFY_GA_Foreign_01`, both spawned at
  z=400 over the landscape, both deleted afterwards — `actor.list {filter:"PWVERIFY"}` -> 0) rather
  than on the shipping `AVE_`/`RUB_` sets, so a guard that did not hold could only have moved my own
  actors. **The fix is in the running binary, not just the source:** a deliberate bad param returns
  `[UNKNOWN_PARAMS] … Valid parameters: [surface, actors, …, expectedMatches, expected_matches, …]`,
  so `expectedMatches` is registered — and it also means that on the pre-fix DLL the tests below
  would have been refused as an unknown param rather than moving anything.
  **(1) The reporter's own call shape is now refused.** `spatial.ground_actors
  {surface:{preset:"landscape"}, prefix:"PWVERIFY_GA_", samples:4, seatPercentile:0.2,
  embedFraction:0.08, detail:"summary"}` — no `expectedMatches`, the exact shape that moved six
  foreign actors — returns `[MISSING_REQUIRED_PARAM] … This selector matched 2. … Run
  spatial.verify_grounding with the SAME selector to see what it matches (it moves nothing), then
  pass expectedMatches:2 - or name the actors outright with 'actors'.` The count is named in the
  refusal, which is the surface the ticket said did not exist at `detail:"summary"`.
  **(2) A wrong count is refused before the first move, and names the over-match.** Same call with
  `expectedMatches: 1` -> `[MATCH_COUNT_MISMATCH] The selector matched 2 actor(s), but
  expectedMatches says 1. NOTHING WAS MOVED.` carrying
  `{"expectedMatches":1,"totalMatches":2,"matchedActors":["PWVERIFY_GA_Mine_01","PWVERIFY_GA_Foreign_01"]}`
  — the 83-vs-89 shape, with the extra actor named instead of a bare count. **"Nothing was moved" was
  checked, not taken on trust:** `actor.get_transform` after both refusals still reported
  `[-9000,1600,400]` and `[-9000,1900,400]`, i.e. the spawn z, unchanged.
  **(3) The undo record now exists for successes.** The honest call (`expectedMatches: 2`,
  `detail:"summary"`) returned `placed:2, failed:0, moved:2` — no failures at all — and still carried
  `movedActors[]` with a `previousTransform` per actor: `PWVERIFY_GA_Mine_01` `{location:{x:-9000,
  y:1600,z:400}, rotation 0, scale 1}` and `PWVERIFY_GA_Foreign_01` `{…y:1900,z:400…}`. Those match
  the transforms I had independently read before the call, and the actor really did move
  (`Mine_01` z 400 -> 42.70), so the receipt is a true pre-move record and not an echo of the
  post-move state. `detail:"summary"` used to carry counts and nothing else; it now carries the undo.
  **Still open elsewhere, deliberately:** encounter 2's `HOLDER_NOT_SEATABLE` refusal for ISM/HISM
  holders is not part of this fix and was NOT tested here — testing it would mean seating a shipping
  HISM holder. It is tracked separately as `F-grounding-holder-not-seatable`, so closing this ticket
  does not close it by association.
