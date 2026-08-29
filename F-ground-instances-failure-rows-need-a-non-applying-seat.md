---
id: F-ground-instances-failure-rows-need-a-non-applying-seat
title: "spatial.ground_instances has no non-mutating seat that reports genuine failures: on apply:false every instance is not-placed by construction, so 'failures' and 'all' become the same setting and dry_run rows consume the whole 256-row cap — the only seat that yields failure-only rows is the one that writes"
status: OPEN
severity: Medium
category: feature
tags: [spatial, ground_instances, detail, dry-run, apply, failure-rows, diagnostics, row-cap, non-mutating, verb-split]
encounters: 1
lastSeen: 2026-08-29
---

# Every dry-run row is a failure row, so no dry-run setting can select the real ones

`spatial.ground_instances`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Spatial/GroundPlacementHandler.cpp:1235`)
merges its verify half into its apply half via `apply:false`, and merges its diagnostics into a
three-valued `detail`. The two interact in a way neither was designed against: **on a dry run there
is no `detail` value that yields failure rows only.**

The mechanism, in three steps.

**1. `bPlaced` is false for every instance on a dry run.** The row selector is

    const bool bWantRow = bPlaced ? bIncludeSuccesses : bIncludeFailures;

`GroundPlacementHandler.cpp:1487`, where `bPlaced = Result.Seat.IsSeated()` (`:1471`) and
`IsSeated()` is `AppliedTransform.IsSet() && Contact.bPass`
(`Handlers/Spatial/GroundPlacementUtils.h:623`). The dry-run branch of `SeatInstance` deliberately
leaves `AppliedTransform` unset — *"Nothing was written, so nothing is claimed: AppliedTransform
stays unset, Seat.WasMoved() stays false"* (`GroundPlacementUtils.cpp:1515-1523`). Correct
behaviour, and it makes `bPlaced` false universally under `apply:false`.

**2. So `bWantRow` collapses to `bIncludeFailures`, and two of the three `detail` values become the
same setting.** `GroundRpcParseDetail` (`GroundPlacementHandler.cpp:586-611`) maps:

- `"summary"` → `bIncludeSuccesses=false, bIncludeFailures=false` (`:590-594`) — **no rows at all**;
- `"failures"` (default) → `false, true` (`:596-600`) — a row for every instance;
- `"all"` → `true, true` (`:602-606`) — a row for every instance, **identical to `"failures"`**,
  because the `bIncludeSuccesses` half is unreachable when nothing is ever placed.

The schema text says as much, in the clause that reads as a caveat and is actually the defect:
*"'failures' (default: counts plus a row per instance that was **not placed - which on a dry run is
every one of them**)"* (`:1324-1328`, emphasis added; rendered verbatim to
`Saved/PinWright/wiki/spatial.ground_instances.md`).

**3. Genuine failures then compete with `dry_run` rows for one bounded array, and lose by position.**
The cap is `constexpr int32 GroundRpcMaxDetailRows = 256` (`:64`, under a comment at `:61-63`
explaining that counts are exact and only rows are bounded). Rows are appended in instance order and
dropped once full:

    if (Rows.Num() >= GroundRpcMaxDetailRows)
    {
        ++DetailRowsDropped;
        continue;
    }

`:1492-1496`. On a 1000-instance scatter with `apply:false`, the first 256 processed instances fill
`results[]` — almost all of them `status:"dry_run"` — and an instance that genuinely fails at index
300 is dropped. The response says `resultsTruncated:true` / `resultsDropped:744` (`:1587-1591`), so
nothing is hidden; it is simply not reachable in that call.

## What IS reachable, stated precisely so the ask is not overclaimed

Genuine failures do occur on a dry run and are **counted honestly**. A measurement failure
(`NoGroundFound`, `GroundHitsAllRejected`, `PartialGroundCoverage`) returns from `SeatInstance`
*before* the `if (!bApply)` branch at `GroundPlacementUtils.cpp:1515` — see the `Pre.bPass` gate at
`:1473-1478` — so such an instance carries a real failure status, not `DryRun`. The counter excludes
only placed and dry-run instances:

    FailedCount += (bPlaced || bDryRun) ? 0 : 1;

`GroundPlacementHandler.cpp:1476`. So `apply:false, detail:"summary"` truthfully answers *how many*
would still fail. **What has no non-mutating seat is the reasons.** `reasonCode`, `reason` and the
per-column `contact` block live only in a row (`:1499-1538`), and the only seat that yields
failure-only rows is `apply:true, detail:"failures"` — which writes.

The consequence for a caller is the ordinary one: they run the dry run to decide whether to run the
real one, get `failed: 42` and no way to see what those 42 hit, and either page the whole scatter in
256-instance windows filtering `status != "dry_run"` client-side, or give up and apply — at which
point the 42 failures are visible and 958 instances have moved.

## Three distinct tickets on this verb's response; here is how they differ

Two adjacent ones are being filed this session by another agent. A triager must be able to tell them
apart, so:

- **`E-ground-instances-two-truncation-flags`** — the row cap *is* announced, but under a second
  field name confusable with the batch-level `truncated` (`:1560` vs `:1589`). About the naming of
  an announcement that exists.
- **`E-ground-provenance-unreachable-at-summary-detail`** — ground provenance rides inside a row, so
  `detail:"summary"` cannot reach it. About a *field* that only a row carries.
- **This ticket** — about a *seat*: there is no configuration of `apply` and `detail` that reports
  genuine failures without writing. It is not the cap being mislabelled (it is labelled) and not one
  field being row-only (every field is); it is that the dry run cannot filter to the rows that
  matter, because on a dry run every instance qualifies as not-placed.

## The right shape of the fix, and the convention it has to be argued against

The board's own rule cuts against another boolean:

> A verb that reads OR mutates based on a boolean (`dryRun`) is wrong — split into two verbs
> (precedent: `blueprint.graph.find_orphaned_nodes` / `delete_orphaned_nodes`).

`agent-conventions.md:51`.

**`F-ism-per-instance-transforms` argued the opposite for this verb specifically, and that argument
must be engaged rather than ignored.** Its proposal says: *"`ground_instances` should **not** be
split into a verify/apply pair. That split exists on the actor side because a name-pattern selector
on a mutating verb needs `expectedMatches` guarding it; instances are addressed by explicit index
against one named component, so there is no pattern hazard and an `apply:false` dry run covers the
verify half."* The shipped handler repeats it in its header comment
(`GroundPlacementHandler.cpp:1228-1233`) and again in the `apply` parameter text (`:1273-1279`).

**That argument is about safety, and it holds. This ticket is about diagnostics, which it did not
consider.** The pattern-scope hazard is genuinely absent — nothing here needs an `expectedMatches`
guard. But the merged verb has a second cost the safety argument never weighed: the dry run's own
success state (`DryRun`) is indistinguishable, to the row selector, from a failure, so the
diagnostic axis loses a degree of freedom the actor-side pair never lost. `spatial.verify_grounding`
has no such problem precisely because it has no dry-run status to confuse with a failure.

**Therefore the proposal is a fourth `detail` value, not a fourth verb and not a new boolean.**
`detail: "would_fail"` — rows only for instances whose status is neither `Seated` nor `DryRun`, i.e.
`bWantRow` gated on `!bPlaced && !bDryRun` rather than on `!bPlaced`. This is the honest reading of
`agent-conventions.md:51`: `detail` is already a diagnostic enum, not a read-or-mutate switch, so
adding a value to it is not the `dryRun`-boolean anti-pattern the rule names — whereas a new
`failuresOnly:true` flag *would* be, and a sibling verb would re-litigate a split
`F-ism-per-instance-transforms` argued down on grounds this ticket agrees with. It also composes:
`apply:true, detail:"would_fail"` is the same rows the current `apply:true, detail:"failures"`
gives, so nothing existing changes meaning.

Two implementation notes: the value must be accepted by `GroundRpcParseDetail` (`:586-611`), which
today rejects anything outside the three with `ERR_INVALID_ARGUMENT` — deliberately, so a typo
cannot silently halve the response; and the two-boolean out-param signature it uses cannot express a
third state, so it needs a third flag or a small enum rather than another `bool&`. If the value is
added to `ground_instances` only, `spatial.ground_actors` and `spatial.verify_grounding` — which share
the same parser (`:870`, `:1113`) — must reject it explicitly rather than inherit it, since neither
has a `DryRun` status for it to mean anything against.

**Severity: Medium, category feature, both argued.**

*Category.* Filed as `feature` rather than `ergonomic` because the ask adds a response mode that does
not exist, not because an existing one is awkwardly named or documented. The two adjacent tickets
above are correctly `E-*` — one is a field name, one is a doc/level mismatch. This one changes what
the verb can be asked for.

*Severity.* Not the rubric's Low — this is not *"a response spill that only forces a `Read`"*; the
information is genuinely absent from the response, not merely inconveniently placed. Not High: no
claim here is false. `failed: N` is exact on a dry run (`:1476`), the cap is announced
(`:1587-1591`), and the schema text states the every-instance-is-a-row behaviour outright
(`:1324-1328`) — a caller is under-served, never deceived. That leaves the rubric's **Medium**:
*"Doable, but only via a documented workaround, a source dive, or many extra calls"*, and it is the
last of those — page the scatter in `limit:256` windows and filter `status != "dry_run"` client-side,
which is ⌈N/256⌉ calls plus client logic to recover something the verb already computed.
**Reach modifier declined in both directions, and named:** `spatial.ground_instances` is not an
every-session verb, so no bump up; but it is not a rare edge path either — it is the only
per-instance grounding verb, and `spatial.ground_actors`' `HOLDER_NOT_SEATABLE` refusal
(`:718-720`) routes every ISM/HISM scatter caller into it, with `apply:false` as the documented
first move. Medium stands unmodified.

## Related

- `E-ground-instances-two-truncation-flags` — same response, the cap's second field name. *(Being
  filed concurrently this session; not present in this working tree at the time of filing.)*
- `E-ground-provenance-unreachable-at-summary-detail` — same response, a row-only field at summary
  detail. *(Same.)*
- `F-ism-per-instance-transforms` (IN-REVIEW, High) — shipped this verb and argued explicitly
  against splitting it. This ticket accepts that argument and proposes a `detail` value instead of a
  sibling verb, for the reasons above.
- `B-ism-undo-record-unsafe` (OPEN, High) — another gap in this verb's response contract
  (`movedInstances[]` rows carry no space marker; `ground_instances` echoes no `space` at all).
- `F-ground-instances-align-to-surface` — the input-side parity gap on the same verb.

## History
- `#1-no-non-writing-failure-seat` `OPEN` reporter — Traced the interaction rather than accepting
  the summary claim, and the finding is narrower and sharper than "failure rows are unreachable
  without mutating": on `apply:false`, `bPlaced` is false for **every** instance because
  `IsSeated()` requires `AppliedTransform.IsSet()` (`GroundPlacementUtils.h:623`) and the dry-run
  branch deliberately leaves it unset (`GroundPlacementUtils.cpp:1515-1523`), so the selector at
  `GroundPlacementHandler.cpp:1487` collapses to `bIncludeFailures` and `detail:"failures"`
  (`:596-600`) and `detail:"all"` (`:602-606`) become **the same setting** — one row per instance,
  competing with genuine failures for the single `GroundRpcMaxDetailRows = 256` cap (`:64`, gate at
  `:1492-1496`) in instance order, so a failure past index 256 is dropped. Also established what
  *does* work, so the ask is not overclaimed: measurement failures return before the `!bApply` branch
  (`GroundPlacementUtils.cpp:1473-1478` vs `:1515`), carry real statuses, and are counted exactly by
  `FailedCount += (bPlaced || bDryRun) ? 0 : 1` (`:1476`) — so `apply:false, detail:"summary"` gives
  an honest failure **count**; what has no non-mutating seat is the **reasons**, which live only in a
  row (`:1499-1538`). Distinguished from the two adjacent tickets being filed this session. Engaged
  `F-ism-per-instance-transforms`' explicit argument against splitting this verb (repeated at
  `:1228-1233` and `:1273-1279`) and accepted it — its pattern-scope reasoning holds — while noting
  it weighed safety and not diagnostics; proposed `detail: "would_fail"` rather than a sibling verb
  or a new boolean, which is the reading of `agent-conventions.md:51` (*"A verb that reads OR mutates
  based on a boolean (`dryRun`) is wrong — split into two verbs"*) that the existing `detail` enum
  already satisfies.
