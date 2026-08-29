---
id: E-ground-instances-two-truncation-flags
title: "`spatial.ground_instances` carries two differently-named truncation flags: the prominent `truncated` is the one documented in the `limit` param and reads false while 21% of the rows are gone, and `resultsTruncated`/`resultsDropped` — the pair that actually covers the row cap — is named in no param description"
status: OPEN
severity: Low
category: ergonomic
tags: [spatial, ground_instances, truncation, response-size, docs, naming, discoverability, detail, convention]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Two flags, one word, and the one a caller reads first is answering a different question

**This is not silent truncation.** The row cap announces itself. The defect is that it announces
itself under a name the documentation never mentions, three lines below a *differently* named flag
that the documentation does mention and that reads `false` at the same moment.

On 324 instances at `detail: "all"`, under the default `limit: 512`, the response carries:

```
truncated:        false     <- documented, prominent, and correct about paging
resultsTruncated: true      <- undocumented, and the one that covers the 68 missing rows
resultsDropped:   68
```

## Confirmed in source

`Handlers/Spatial/GroundPlacementHandler.cpp`:

- Row cap constant `GroundRpcMaxDetailRows = 256` at `:64`, with its rationale at `:61-63`:
  *"Ceiling on per-actor rows echoed in the response. Counts are always exact and uncapped; only the
  detail rows are bounded, the same split actor.spawn_batch uses for skipped[]
  (SpawnBatchHandler.cpp:248-261)."*
- Drop site `:1492-1496` — `if (Rows.Num() >= GroundRpcMaxDetailRows) { ++DetailRowsDropped; continue; }`.
- `truncated` at `:1560` is `Processed < FMath::Max(Indices.Num() - Offset, 0)`. It measures the
  **processing batch** — `limit` and `offset` — so on 324 instances under a 512 default it is
  correctly `false`. It is not lying; it is answering the question it was built for.
- `resultsTruncated` / `resultsDropped` at `:1587-1591` (`:1589` and `:1590`), emitted only when
  `DetailRowsDropped > 0`.

## Where the documentation points, and where it does not

- The `limit` param text (`:1319-1321`) is where a caller reads about truncation: *"Maximum instances
  to process in this call (1-5000). Pair with offset to page a larger scatter; instanceCount always
  reports the component's true total."* That is what `truncated` at `:1560` reports.
- The `detail` param text (`:1324-1328`) is the only place the row cap is mentioned, and it names no
  field: *"Row arrays are capped; counts never are"* (`:1326-1327`). A caller who reads that sentence
  and then goes looking for the flag finds `truncated`, which is the wrong one, and it says `false`.

So the failure mode is a caller who checks the documented flag, gets `false`, and concludes the row
array is complete. The correct flag is present in the same object and is discoverable only by
reading every key of a response that has already been truncated.

## The convention is inconsistent across three sibling verbs

Three field names across two concepts, none of them cross-referenced:

| verb | field | covers |
|---|---|---|
| `spatial.ground_instances` | `truncated` (`GroundPlacementHandler.cpp:1560`) | the paging batch (`limit`/`offset`) |
| `spatial.ground_instances` | `resultsTruncated` + `resultsDropped` (`:1589-1590`) | the 256-row detail cap |
| `foliage.add_instances` | `skippedTruncated` (`FoliageHandler.cpp:1248`) | the 32-row `skipped[]` cap |
| `actor.spawn_batch` | `skippedTruncated` (`SpawnBatchHandler.cpp:368`) | the 32-row `skipped[]` cap |

**Correction to the shape this was first reported in:** `actor.spawn_batch` and
`foliage.add_instances` use the *same* spelling — `skippedTruncated` — so it is two spellings for the
row-cap concept, not three, and the two `skipped` verbs are already consistent with each other. The
outlier is `ground_instances`, which invents `resultsTruncated` for the identical idea and is also
the only one of the three to overload the bare word `truncated` for a second, different meaning
in the same object.

One further asymmetry worth a line: `resultsDropped` publishes the drop count directly, while
`skippedTruncated` is a bare boolean — recoverable only because `skippedCount` carries the true
total beside it. The `ground_instances` shape is the better one; it is just named least like its
siblings.

**A one-convention ticket would cover all four rows of that table. Do not file one** — board policy
is against umbrellas. It is named here as an option for whoever picks this up, because the cheapest
honest fix to *this* ticket (rename or alias `resultsTruncated` -> `rowsTruncated` and document it)
is also the point at which the convention question has to be answered anyway.

## Ask

Smallest sufficient change, in order:

1. **Name the field in the `detail` param text.** `:1326-1327` currently says row arrays are capped
   without saying what to read. Naming `resultsTruncated` / `resultsDropped` there closes the
   discoverability half at zero risk.
2. **Disambiguate `truncated` in the `limit` param text** (`:1319-1321`) — say that it covers paging
   only and points at the other pair for rows. A caller who reads one param description should not
   be able to reach the wrong conclusion.
3. Optionally, rename `truncated` -> `pageTruncated` (keeping `truncated` as an alias), which is the
   only change that removes the trap rather than documenting it. This is a wire change and should not
   be done for `ground_instances` alone.

## Dedup

There is no existing "cap reported as complete" family on this board. The ~20 `E-*-no-limit-spills`
tickets (`E-actor-list-no-limit-spills`, `E-asset-list-no-projection-spills`,
`E-console-search-default-limit-spills`, …) are the **opposite** complaint — no cap at all, so the
response spills. `E-widget-describe-slot-truncation` is the nearest in spirit and is about a
different verb and a different field.

## Cross-links

- `E-spill-threshold-measured-post-wrap` (OPEN, Medium) — why the caps exist at all, and why they
  bite ~2.35x sooner than the 10,000-char budget the comment at `:61-63` is reasoning against.
- `B-ism-undo-record-unsafe` (OPEN, High) — its sixth-gap encounter covers the *uncapped*
  `movedInstances[]` on this same verb. The two are the same design question from opposite ends: one
  array is capped and misnamed, the sibling array beside it is uncapped and deliberately so.
- `E-ground-provenance-unreachable-at-summary-detail` — compounding: `detail: "all"` is the only
  level that emits provenance, and it is the level this cap applies to.

## Severity

**Low.** Impact class: *pure friction — docs, discoverability, naming*. Nothing here reports wrong
data. `truncated: false` is a true statement about paging, `resultsTruncated: true` is a true
statement about rows, and both are present in every affected response. The cost is a caller reading
the documented field, drawing the wrong conclusion, and needing a source dive or a full key dump to
find the right one — friction, not a lie.

**It is deliberately not rated High.** The High band is *silent* false-success or wrong data. This is
neither silent (the flag is emitted) nor wrong (both flags are accurate). Rating it higher on the
strength of "a caller could be misled" would inflate every naming defect into a correctness defect.

**Reach modifier declined.** `detail: "all"` is not the default, so the strict reading argues for a
downward bump — but `Low` is the floor and cannot be bumped below it, and the misleading pairing sits
in the `limit`/`detail` params that every call to this verb reads. So Low with no adjustment, not
Low-by-default: the rubric's "never default to Low" is met by argument, not by omission.

## History
- `#1-two-flags-one-word` `OPEN` reporter — Filed after a first report of *silent* truncation at
  `detail: "all"` was checked and found wrong: the cap is announced, by `resultsTruncated` /
  `resultsDropped` (`GroundPlacementHandler.cpp:1589-1590`). Re-derived: cap constant `:64`, rationale
  `:61-63`, drop site `:1492-1496`, `truncated` `:1560` measuring the paging batch, `limit` text
  `:1319-1321`, `detail` text `:1324-1328` naming no field. Corrected a second time against the
  report's "three spellings" claim: `actor.spawn_batch` (`SpawnBatchHandler.cpp:368`) and
  `foliage.add_instances` (`FoliageHandler.cpp:1248`) use the same `skippedTruncated`, so it is two
  spellings for one concept plus an overloaded bare `truncated`. Umbrella deliberately not filed, per
  board policy; the convention option is named for the fixer instead.
