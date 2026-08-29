---
id: E-bounds-plane-fallback-has-no-batch-level-count
title: "The mesh-profile → bounds-plane fallback is honest per row and invisible in the batch: `seat.undersideModel` / `criteria.undersideModel` echo the REQUESTED model with no count of rows that fell back, so a batch echo can say `mesh` while every row measured a flat AABB plane — and on those rows `undersideReliefCm` is forced to exactly 0 and `maxColumnClearanceCm` collapses onto `maxGapCm`, two shape numbers that read as measured-and-flat rather than not-measured"
status: OPEN
severity: Medium
category: ergonomic
tags: [spatial, verify_grounding, ground_actors, ground_instances, underside-model, bounds-plane, fallback, batch-echo, zero-is-absence, readback, placement, level-building]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Six rows said "I fell back"; the batch said `mesh`

Measured live on `EAContentExamples58` (UE 5.8): all six `S_Forest_Rock_Shelf` actors in one batch
fell back from the mesh-profile underside model to the flat bounds plane. Each row said so. The
batch echo did not, and a caller reading the echo would take the batch as mesh-profile measured.

## Correction to the report this was filed from: the fallback is NOT silent per row

Stated first, because the brief that produced this ticket had it as a silent fallback and that is
wrong at HEAD. The per-row honesty exists and is deliberate. `GroundRpcContactObject`
(`Source/PinWright/Private/Handlers/Spatial/GroundPlacementHandler.cpp`) emits, after the
`bMeasured` early return at `:490-493`:

```cpp
// GroundPlacementHandler.cpp:500-506
Obj->SetStringField(TEXT("undersideModel"),
    Report.UndersideModel == GroundPlacement::EUndersideModel::BoundsPlane
        ? TEXT("bounds_plane") : TEXT("mesh"));
if (Report.bUsedBoundsPlaneFallback)
{
    Obj->SetBoolField(TEXT("boundsPlaneFallback"), true);
}
```

`undersideModel` is unconditional on every measured row; `boundsPlaneFallback: true` is added
whenever the flag is set. And the code that sets it says so in as many words
(`GroundPlacementUtils.cpp:1053-1057`):

> Mesh-profile fallback: the underside geometry answered no geometry query anywhere (collision
> disabled, no body instance, nothing cooked, or — for an instance — no actor whose own components
> could be asked). Fall back to the flat AABB plane and **SAY SO** — the numbers below mean
> something different under that model, and a caller that cannot tell the two apart is back to
> trusting an unverifiable result.

That is exactly right and this ticket does not weaken it. **The gap is that the "say so" reaches the
row and not the batch, and that two numbers on the fallback row are zeroed rather than omitted.**

## Mechanism 1 — the batch echoes the request, not the outcome

The fallback trigger, `GroundPlacementUtils.cpp:1058-1071`:

```cpp
int32 GeometryColumns = 0;
for (const FGroundColumn& Column : Columns)
{
    GeometryColumns += Column.bHasActorGeometry ? 1 : 0;
}
if (UndersideModel == EUndersideModel::MeshProfile && GeometryColumns == 0)
{
    Report.bUsedBoundsPlaneFallback = true;
    for (FGroundColumn& Column : Columns)
    {
        Column.bHasActorGeometry = true;
        Column.UndersideZ = BottomZ;
    }
}
```

It is per actor, decided inside the measurement. The batch echoes, however, are written once from
the parsed **config**:

- `spatial.ground_actors` — `seat.undersideModel`, `GroundPlacementHandler.cpp:990-992`, from
  `Config.UndersideModel`.
- `spatial.verify_grounding` — `criteria.undersideModel`, `:1201-1202`, from the parsed `Model`.

Neither consults any `Report`. So a batch in which every row fell back still echoes `mesh`, and
there is no `boundsPlaneFallbackCount` anywhere in the response. At the **default**
`detail: "failures"` (`GroundRpcParseDetail`, `:589`) a passing row emits no `contact` object at all
(`:1162-1163`), so for a batch that passes, the per-row honesty is not merely buried — it is not in
the response, and the echo saying `mesh` is the only statement about the underside model the caller
receives.

Out of scope, named only to keep the boundary clear: `spatial.ground_instances` hardcodes
`SeatEcho->SetStringField(TEXT("undersideModel"), TEXT("bounds_plane"))` at `:1573`, under a comment
at `:1570-1572` explaining that an instance's underside cannot be probed per column because
`UInstancedStaticMeshComponent::LineTraceComponent` answers from every instance body at once. That
echo is honest — it states what the verb always does — and is **not** part of this ticket.

## Mechanism 2 — two shape numbers become zeros that read as measurements

Forcing every `Column.UndersideZ = BottomZ` makes the underside a single plane, and both shape
fields are defined as spreads over exactly that set. With `Clearance() = UndersideZ - GroundZ`
(`GroundPlacementUtils.h:244`):

- `UndersideReliefCm = MaxUnderside - MinUnderside` (`GroundPlacementUtils.cpp:578`) →
  `BottomZ - BottomZ` = **exactly 0**.
- `MaxColumnClearanceCm = max_i(BottomZ - GroundZ_i)` = `BottomZ - min_i GroundZ_i`, while
  `MaxGapCm = MinUnderside - MinGround` = `BottomZ - MinGround` (`:547`, `:557`, `:577`) → the two
  are **identical**, so the documented invariant `MinGapCm <= MaxGapCm <= MaxColumnClearanceCm`
  (`GroundPlacementUtils.h:428`) silently degenerates to an equality.

Both fields are then serialized like any other measurement (`GroundPlacementHandler.cpp:517-518`).
Their own declarations say what a caller is supposed to read out of them, and neither statement is
true on a fallback row — `GroundPlacementUtils.h:438-440`:

> Highest minus lowest underside sample across supported columns: how far the actor's own underside
> is from being a plane. **Large relief with a small `MaxGapCm` is a shaped actor that is properly
> seated** — exactly the case that used to read as a placement bug.

A shaped rock shelf that reports `undersideReliefCm: 0` is not being described as unmeasured. It is
being described as flat-bottomed, which is the opposite of what it is, and it is the discriminator
`B-verify-grounding-maxgap-false-fail` built these two fields to provide.

## The precedent for the fix is four lines up, in the same function

`GroundRpcContactObject` already knows that a zero which is really an absence reads as a result, and
already applies the rule — to the gap numbers, one field group above these two
(`GroundPlacementHandler.cpp:508-511`):

```cpp
// The gap numbers only mean something when at least one column found ground.
if (Report.SupportedColumns > 0)
{
    Obj->SetNumberField(TEXT("maxGapCm"), Report.MaxGapCm);
```

The same sentence is true of the two shape numbers when no column found geometry. The file applies
the rule to one set of fields and not the other, in the same `if` block, thirteen lines apart.

## Ask

1. **`boundsPlaneFallbackCount` beside the batch echo** — on `spatial.ground_actors`' `seat` and
   `spatial.verify_grounding`'s `criteria`, counted over the reports the verb already builds in its
   own loop (`:1154-1180` for `verify_grounding`). It costs one integer and one increment; the
   information is already in `Report.bUsedBoundsPlaneFallback` per actor and is currently discarded
   at batch level. It must be a **count**, not a bool, so a partially-degraded batch is
   distinguishable from a wholly-degraded one — and it must not be governed by `detail`, for the
   same reason the undo logs are not (`:996`, `:1576`): it describes what the verb did, not how
   verbose it was.
2. **Omit `undersideReliefCm` and `maxColumnClearanceCm` on a fallback row** rather than emitting
   `0` and a duplicate of `maxGapCm`, following the `SupportedColumns > 0` precedent above. This is
   the sharper half: rung 1 tells a caller to go look, rung 2 removes the thing that stops them
   looking. Absence of a field is a question; a `0` is an answer.

The two are independent — either can land alone — but shipping 1 without 2 leaves the batch pointing
at rows whose numbers still misdescribe themselves.

**Cost, stated rather than skipped.** Rung 2 is a **wire-shape change on a passing path**: any
caller or test that reads `undersideReliefCm` unconditionally breaks on a fallback row. That is the
right trade by this file's own precedent, but it is not free, and a fixer should grep the tests in
`Private/Tests/Spatial/TestGroundPlacement.cpp` (which assert on both fields, e.g. the
`CurvedUndersideStillPasses` fixture that requires both `> 10`) before changing it. Rung 1 is purely
additive.

## Severity: Medium

**Impact class is the README's Medium band — "a readback omits a field and forces a fallback".** The
information exists and is reachable: every measured row carries `undersideModel` and, when it
applies, `boundsPlaneFallback: true` (`:500-506`), so a caller who passes `detail: "all"` and reads
per row has the complete truth. What the response omits is the aggregate, which forces a per-row
read to answer a batch-level question — the definition of the band.

**The argument for High, weighed and declined.** Mechanism 2 is genuinely "wrong data on a normal
path": `undersideReliefCm: 0` asserts flat where nothing was measured, and
`maxColumnClearanceCm == maxGapCm` asserts an agreement the geometry never produced. High requires
that the caller *trusts a result that is a lie* — and on the same row, in the same object, sits
`boundsPlaneFallback: true`, which explains both zeros. The row is self-describing; the zeros are
misleading but not unaccompanied. That is what separates this from
`B-ground-probe-hits-hull-not-render`, where the response contained no field that could contradict
the number. Declined for that reason, and only for that reason: if a future change ever emits those
two fields without the fallback flag beside them, this becomes High immediately.

**`Critical` is unreachable** — it is impact-class only (crash, or a write that corrupts or loses
asset data) and neither applies. **`High or Medium: hard blocker with no workaround` does not
apply** — `detail: "all"` plus a per-row read is a working workaround for the batch question.

**Reach modifier: neutral — I decline the every-session bump UP.** Both grounding verbs run in most
level-building sessions, which argues for it. Against it: the fallback path is narrow, requiring
`GeometryColumns == 0` — *every* column's underside probe missing (`:1063`), i.e. collision
disabled, no body instance, or nothing cooked. Common method, uncommon path, so the two cancel; same
split held neutral by `B-ground-probe-hits-hull-not-render` `#1`. **Severity `Medium`, reach
neutral.**

## Same shape as

`B-verify-grounding-maxgap-false-fail`, `B-ground-probe-hits-hull-not-render` — the call succeeds,
every number it reports is correct for what it measured, and the caller is misled because the
response does not say what was measured. Fullest statement of the class in
`B-foliage-paint-does-no-ground-projection` `## Same shape as`; referenced, not restated.

The local variant worth naming separately, since it recurs across this file: **a zero that means
"not measured" is indistinguishable from a zero that means "measured, and it was zero"**, and this
codebase has already decided that question once, correctly, at `GroundPlacementHandler.cpp:508-511`
and at `GroundPlacementUtils.cpp:854-856` (where `-1` for "no body setup" is omitted rather than
forged into a `0`, under a comment saying a `0` there would read as the opposite claim). This is the
third field group in the same response object where the rule applies and the first where it was not
applied.

## Cross-links

- `B-verify-grounding-maxgap-false-fail` (DONE, High) — created both affected fields in `#3` and
  defined `undersideReliefCm` as "the size of the shape term". This ticket is a coverage gap inside
  that fix: the decomposition is correct and its `#4` live verification stands; what was not
  considered is what the two new fields mean when the underside was never probed.
- `B-ground-probe-hits-hull-not-render` (IN-REVIEW → DONE this session) — the neighbouring
  "response cannot say what it measured" defect on the same response object, and the source of the
  `-1`-is-not-`0` precedent cited above.
- `E-grounding-coverage-cannot-catch-a-single-point-rest` — filed alongside this one, and the direct
  complement: the fallback here is all-or-nothing at `GeometryColumns == 0`, so a footprint where
  *one* of nine columns found geometry gets neither the fallback nor its flag, and lands in that
  ticket's degenerate-`actorColumns` case instead. A fixer taking either should read both, because
  the honest treatment of "almost no geometry columns" is unowned between them.
- `F-grounding-holder-not-seatable` (IN-REVIEW) — the instance path is named in the fallback
  comment's own list of causes ("for an instance — no actor whose own components could be asked"),
  so a holder refusal would remove one population of fallback rows without addressing the echo.

## History
- `#1-batch-echo-reports-the-request` `OPEN` reporter — Filed 2026-08-29 from live measurement on
  `EAContentExamples58` (UE 5.8): six `S_Forest_Rock_Shelf` actors, every row on the bounds-plane
  fallback, batch echo `mesh`. Every mechanism claim re-derived by `grep -n` / `sed -n` at HEAD in
  this tree. **The report this was filed from described the fallback as silent per row; that is
  false at HEAD and is corrected in its own section above rather than quietly dropped** — the per-row
  disclosure at `GroundPlacementHandler.cpp:500-506` and the "SAY SO" comment at
  `GroundPlacementUtils.cpp:1053-1057` are credited, and the ticket is narrowed to the batch echo
  and the two zeroed fields. **Dedup sweep before filing.** `B-verify-grounding-maxgap-false-fail`
  (DONE) created the two fields but its defect and fix are the float/shape decomposition; nothing in
  it concerns the fallback, and `undersideModel` appears in it nowhere.
  `B-ground-probe-hits-hull-not-render` is about which collision representation answered, an
  orthogonal axis on the same object. `F-grounding-holder-not-seatable` is about refusing a holder
  subject. A board-wide grep finds no ticket mentioning `undersideModel`, `boundsPlaneFallback`,
  `undersideReliefCm` or `maxColumnClearanceCm` as a defect. Filed new. **Evidence:** the per-actor
  trigger at `:1058-1071` against batch echoes written from config at `:990-992` and `:1201-1202`;
  the arithmetic collapse derived from `Clearance()` (`GroundPlacementUtils.h:244`) with every
  `UndersideZ` forced to `BottomZ`, giving `undersideReliefCm == 0` and
  `maxColumnClearanceCm == maxGapCm` exactly, which turns the documented invariant at
  `GroundPlacementUtils.h:428` into an equality; and the in-file omission precedent at
  `GroundPlacementHandler.cpp:508-511`. Severity `Medium` argued above against the rubric, with the
  High argument for mechanism 2 stated and declined (the fallback flag sits on the same row), the
  condition under which that decline would reverse named, and the declined every-session reach bump
  named. **No umbrella:** two additive changes on two verbs; the recurring-class statement is
  referenced at `B-foliage-paint-does-no-ground-projection`. **Not RPC-verified this pass** — the
  batch echo's contents were established from source; the six-actor observation came from the
  session's own measurement and no fresh `verify_grounding` call was made here.
  **Concurrency note:** `B-ground-probe-hits-hull-not-render.md` was read-only — cited, never
  modified.
