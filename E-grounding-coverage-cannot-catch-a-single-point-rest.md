---
id: E-grounding-coverage-cannot-catch-a-single-point-rest
title: "`coverage` is `supportedColumns / actorColumns`, so at `actorColumns: 1` it is exactly 1 whatever the seat is, and `minContactPoints` — the only absolute guard — defaults to 1; a shelf actor resting on a single point reports `coverage: 1, contactPoints: 1` and passes, which is the failure these verbs exist to prevent presenting as a perfect seat"
status: OPEN
severity: Medium
category: ergonomic
tags: [spatial, verify_grounding, ground_actors, ground_instances, coverage, contact-points, actor-columns, thresholds, single-point-rest, footprint-columns, placement, level-building, unsound-workaround]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# A ratio over a denominator of 1 is not a measurement, and it is the field callers were told to gate on

Measured live on `EAContentExamples58` (UE 5.8): a shelf actor
(`S_Forest_Rock_Shelf_wjsqdii_lod0_Var1`) reported

```
actorColumns: 1, supportedColumns: 1, coverage: 1, contactPoints: 1, groundSpreadCm: 0
```

and passed every criterion. That is a **single-point rest** — the balanced-boulder case these verbs
were written to catch — arriving as a flawless row.

## Why no threshold can express "enough columns"

**`coverage` is a ratio and its denominator is the thing that collapsed.**
`Source/PinWright/Private/Handlers/Spatial/GroundPlacementUtils.cpp:611-613`:

```cpp
Report.Coverage = (Report.ActorColumns > 0)
    ? (static_cast<double>(Report.SupportedColumns) / static_cast<double>(Report.ActorColumns))
    : 0.0;
```

At `ActorColumns == 1` the value is binary: `1` if that one column found ground, `0` if it did not.
The `0` branch never reaches the coverage gate — `EvaluateContact` exits earlier on
`GROUND_NOT_FOUND` when nothing was supported (`:725-731`). **So among rows that reach the coverage
check, `coverage` at `actorColumns: 1` is exactly `1.0`, always.**

That makes `minCoverage` structurally unable to reject it. The check is
`if (Report.Coverage + UE_KINDA_SMALL_NUMBER < Thresholds.MinCoverage)` (`:734`); the default is
`constexpr double DefaultMinCoverage = 0.5` (`GroundPlacementUtils.h:97`, threshold field `:285`),
and the parameter is **clamped to `0.0-1.0`** at all three read sites
(`GroundPlacementHandler.cpp:916-918`, `:1138-1140`, `:1439-1441`). A value that could reject a
coverage of exactly `1.0` would have to exceed `1.0`, and the clamp forbids it. This is the same
shape as `B-verify-grounding-maxgap-false-fail` `#1`'s "no value of `maxGap` works" argument: the
knob exists, is tunable, and cannot reach the case, **because the compared number is not a property
of the seating.**

`minCoverage` therefore means "enough of however few columns there were", never "enough columns".

**The only absolute guard defaults to 1.** `minContactPoints` is the one threshold counting
something rather than a fraction (`:745`, `if (Report.ContactPoints < Thresholds.MinContactPoints)`,
where a contact point is a column whose clearance is within `ContactToleranceCm`, `:564-568`). Its
default is `1` at all four sites, verified individually:

| site | text |
|---|---|
| `GroundPlacementHandler.cpp:919-920` (`ground_actors`) | `Ctx.GetInt(TEXT("minContactPoints"), Ctx.GetInt(TEXT("min_contact_points"), 1))` |
| `GroundPlacementHandler.cpp:1141-1142` (`verify_grounding`) | same expression |
| `GroundPlacementHandler.cpp:1442-1443` (`ground_instances`) | same expression |
| `GroundPlacementUtils.h:286` (struct) | `int32 MinContactPoints = 1;` |

The three parameter declarations (`:824-826`, `:1083-1086`, `:1307-1309`) each publish `"1"` as the
default string as well. So `contactPoints: 1` clears the only bar that could have caught it.

**Negative, checked by grep across the whole plugin: there is no `minSupportedColumns`.**
`grep -rn "minSupportedColumns\|MinSupportedColumns\|min_supported_columns"` over
`X:/src/unreal/EAContentExamples58/Plugins/PinWright/` returns **nothing** (exit 1). No absolute
column threshold exists on any of the three verbs, in any spelling.

## The published workaround is unsound in exactly the case it was written to guard

This is the sharp end. `B-verify-grounding-maxgap-false-fail` (DONE, High) closes with a Workaround
section that reads, verbatim:

> Ignore `pass` and `maxGapCm` for anything that is not flat-bottomed. Gate on `coverage`,
> `contactPoints >= 3`, and `penetrationCm` within the intended embed.

At `actorColumns: 1` both named gates fail the caller:

- **`coverage`** is structurally `1.0`, as derived above. A caller gating on it is comparing against
  a constant.
- **`contactPoints >= 3`** is sound advice and **is not the shipped default** — the default is `1`
  (table above). A caller who reads that sentence as a description of the verb rather than as an
  instruction to pass `minContactPoints: 3` gets no protection at all, and the sentence does not
  name the parameter.

**State plainly what this is and is not.** This is a gap in the *advice*, not a defect in that
ticket's fix. `#3`/`#4` of that ticket decomposed the gap into a float term and a shape term and
verified the result live (13 columns, `passed: 5 → 12`); nothing here touches that, and the fix
holds. What the Workaround section did was redirect callers off `pass` and onto two fields, and
neither field was checked against the degenerate footprint. A DONE High ticket's Workaround is the
most-trusted prose on this board precisely because it was verified — which is why an unsound line in
one is worth its own ticket rather than a correction buried in a history entry.

## `actorColumns: 1` is itself the finding, and nothing reports it as abnormal

`samples` defaults to `3`, i.e. a `3x3 = 9` column grid, and the parameter's own description
already names this failure mode (`GroundPlacementHandler.cpp:782-786`):

> Footprint sampling grid per axis: 3 means a 3x3 = 9 column grid over the actor's bounds. **1
> degrades to a single centre column (the old pivot-probe behaviour, and the reason a boulder ends
> up balanced on one point).** Clamped to 1-9.

So the verb knows what a one-column footprint means. It just does not check for it. There are two
ways to arrive at `actorColumns: 1` and they want different responses:

1. **`samples: 1` was passed.** The caller asked for the pivot probe and got it. Worth an echo, not
   an alarm.
2. **`samples: 3` (or more) and the footprint collapsed.** `ActorColumns` counts columns with
   `bHasActorGeometry` (`:505-508`), which under `EUndersideModel::MeshProfile` is whatever
   `GroundProbeUndersideZ` answered per column (`:62`, called at `:993-994`). **Eight of nine
   underside probes missing is a real defect and is reported as nothing.** Note the interaction with
   the bounds-plane fallback: it triggers only at `GeometryColumns == 0` (`:1063`), so 1-of-9 gets
   neither the fallback nor its `boundsPlaneFallback: true` flag — the honesty flag exists for the
   total miss and not for the near-total one.

The response cannot currently distinguish those two, because `samples` is echoed on
`verify_grounding` (`criteria.samples`) but nothing compares it to `actorColumns`.

## Ask

1. **`minSupportedColumns`** — an absolute threshold, not a ratio, defaulting above `1`, beside the
   existing `minCoverage`/`minContactPoints`. This is the one thing no current parameter can
   express. Note that `2` is the smallest defensible default and that raising it is a behaviour
   change for anyone deliberately using `samples: 1`, so the default has to be chosen against the
   `samples` value rather than in isolation.
2. **Refuse to report `coverage` at all below an `actorColumns` floor**, rather than reporting a
   ratio that carries no information. The precedent is already in the response writer, four lines
   above where `coverage` is emitted: `GroundPlacementHandler.cpp:508-511` guards the gap numbers on
   `if (Report.SupportedColumns > 0)` under the comment *"The gap numbers only mean something when
   at least one column found ground."* The same sentence is true of a ratio over one column.
3. **Report the footprint collapse.** Whichever of the two routes above produced it, `actorColumns`
   being far below `samples^2` is a fact about the measurement's quality and belongs beside the
   measurement — cheapest form is a `warning` when `actorColumns` is 1 (or below some fraction of
   `samples^2`) while `samples > 1`, which needs no new probing.

Rungs 1 and 3 are independent; a fixer could land either alone. Rung 2 is a wire-shape change and
should not ship without 1, or callers lose a field and gain no gate.

## Severity: Medium

**Impact class is the README's Medium band — "doable, but only via a documented workaround … or a
readback omits a field and forces a fallback".** Every number in the response is arithmetically
correct; the verb applied the bar it was given and **echoed that bar back** in the same response
(`criteria.minContactPoints`, `criteria.minCoverage`, `GroundPlacementHandler.cpp:1198-1199`,
emitted at every `detail` level). A caller can fix their own exposure in one argument
(`minContactPoints: 3`). The defect is that the verb offers no threshold that can express "enough
columns" and no signal that the footprint collapsed, so the check has to be written outside the verb
that exists to do it.

**Two arguments for High, both weighed and declined.**

- *The unsound published workaround.* Real, and it is the strongest part of this ticket — but it is
  a defect in a ticket's prose, and severity here is a property of the tool's response. Following
  bad advice is how this got found; it is not what makes the tool wrong. It raises urgency, and
  urgency on this board is `encounters`, which the README states explicitly is never a severity
  input.
- *The default `detail: "failures"` hides the row entirely.* `GroundRpcParseDetail` defaults to
  `"failures"` (`GroundPlacementHandler.cpp:589`) and a passing row emits no `contact` object at all
  (`:1162-1163`), so at the default a batch of six single-point rests returns only `passed: 6` and
  `actorColumns` is not in the response. That is the closest this comes to High's "silent
  false-success". Declined because `criteria` **is** emitted at every detail level and states the
  bar that was applied, so the response is not claiming more than it checked — the row's absence is
  a verbosity choice, not an assertion. It does mean the ask's rung 3 warning must be a **batch**
  signal, not only a row field, or the fix inherits this same hole.

**`High or Medium: hard blocker with no workaround` does not apply**: `minContactPoints: 3` is a
working, one-argument workaround that fully closes the reported case.

**Reach modifier: neutral — I decline the every-session bump UP.** `verify_grounding` is the
published proof that a batch is seated and runs in most level-building sessions, which argues for
the bump. Against it: the `actorColumns == 1` path is narrow — it needs `samples: 1` or a footprint
whose underside probe answered in one column — so the common **method** carries an uncommon **path**.
Same shape as `B-ground-probe-hits-hull-not-render` `#1`, which held reach neutral on exactly this
split (common verbs, divergence only on authored architecture). **Severity `Medium`, reach neutral.**

## Same shape as

`B-verify-grounding-maxgap-false-fail`, `B-ground-probe-hits-hull-not-render` — the call succeeds,
every number it reports is correct, and the verdict is wrong because the deciding quantity was never
compared. Fullest statement of the class in `B-foliage-paint-does-no-ground-projection`
`## Same shape as`; referenced, not restated.

Sharper local kinship: this and `B-verify-grounding-maxgap-false-fail` are the **same arithmetic
mistake in opposite directions**. There, a max over the wrong set made a good seat fail; here, a
ratio over a degenerate set makes a bad seat pass. Both were invisible because the number involved
was correct.

## Cross-links

- `B-verify-grounding-maxgap-false-fail` (DONE, High) — the source of the contradicted Workaround
  sentence, and the precedent for the "no value of the knob works" argument. Its fix is not in
  question here.
- `F-grounding-holder-not-seatable` (IN-REVIEW) — the neighbouring case where a subject has no
  meaningful underside at all. A holder measured as a single prop is another route to a degenerate
  footprint; whoever fixes either should read the other, since a `HOLDER_NOT_SEATABLE` refusal would
  remove one source of `actorColumns: 1` without addressing the threshold gap.
- `E-ground-preset-excludes-only-foliage-actors` (IN-REVIEW) — same response object, and its
  `surfaceComponents[]` is the precedent for adding a diagnostic that explains a measurement rather
  than gating on it.
- `E-bounds-plane-fallback-has-no-batch-level-count` — filed alongside this one; the fallback's
  all-or-nothing trigger at `GeometryColumns == 0` is why a 1-of-9 collapse gets neither treatment.

## History
- `#1-coverage-ratio-degenerates-at-one-column` `OPEN` reporter — Filed 2026-08-29 from live
  measurement on `EAContentExamples58` (UE 5.8), `S_Forest_Rock_Shelf_wjsqdii_lod0_Var1`; every
  mechanism claim re-derived by `grep -n` / `sed -n` at HEAD in this tree, not carried from the
  report. **Dedup sweep before filing.** `B-verify-grounding-maxgap-false-fail` (DONE) is the
  nearest neighbour and does not own this: its defect and its fix are both about `maxGapCm` and the
  float/shape decomposition, and its `#4` verification is intact — this ticket cites its *Workaround
  prose* as contradicted, explicitly not its fix. `F-grounding-holder-not-seatable` (IN-REVIEW) is
  about refusing an ISM/HISM holder, a different subject class. `B-ground-probe-hits-hull-not-render`
  is about which surface answered, not how many columns were sampled. `grep` over the whole board
  finds no ticket mentioning `actorColumns`, `supportedColumns`, `minCoverage` or
  `minSupportedColumns` as a defect. Filed new. **Evidence:** the coverage formula at `:611-613`
  with its `GROUND_NOT_FOUND` early exit at `:725-731` (together making `coverage` exactly `1.0` on
  every row that reaches the gate at `actorColumns: 1`); the `0.0-1.0` clamp at `:916-918`,
  `:1138-1140`, `:1439-1441` (making `minCoverage` structurally unable to reject it); all four
  `minContactPoints` default-1 sites verified individually; and the whole-plugin grep for
  `minSupportedColumns` returning nothing. Severity `Medium` argued above against the rubric, with
  both High arguments — the unsound published workaround and the default-detail row suppression —
  stated and declined with reasons, and the declined every-session reach bump named. **No umbrella:**
  this is one missing threshold on three verbs; the recurring-class statement is referenced at
  `B-foliage-paint-does-no-ground-projection`. **Correction to the framing this was filed from:**
  `coverage` is not "1 whenever anything at all was found" in general — it is binary at
  `actorColumns: 1` (`1` or `0`), and the `0` case is unreachable at the coverage gate because
  `GROUND_NOT_FOUND` fires first. The effect is as reported; the mechanism is one branch narrower.
  **Concurrency note:** `B-verify-grounding-maxgap-false-fail.md` was read-only here — quoted, never
  modified.
