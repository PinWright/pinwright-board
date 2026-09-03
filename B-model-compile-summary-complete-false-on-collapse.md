---
id: B-model-compile-summary-complete-false-on-collapse
title: "model.compile's diagnosticSummary.complete goes false whenever any diagnostic repeats, even though collapse suppresses nothing - the wiki calls it 'the decisive flag' and gating on it is a false alarm"
status: OPEN
severity: Low
category: bug
tags: [model, model.compile, diagnostics, diagnosticSummary, collapse, gate, misleading-field]
encounters: 1
lastSeen: 2026-09-03T04:20:00Z
---

# `complete` measures emission, but it is documented as measuring loss

`ModelCompileHandler.cpp:567` computes it as

```cpp
Summary->SetBoolField(TEXT("complete"), Values.Num() == Diagnostics.Num());
```

`Values` is the emitted array. `collapse` is ON by default and folds repeats of one diagnostic into a
single entry carrying `occurrences` and `occurrenceSites`, so a document with any repeated warning
has `emitted < total` and reports `complete: false` while **nothing was dropped** - every occurrence
is still addressable through `occurrenceSites`.

The summary already publishes the two fields that separate the cases, and
`Saved/PinWright/wiki/model.compile.md:37` publishes the identities:

    total == represented + suppressedBySeverity + suppressedByLimit
    represented == emitted + collapsedOccurrences

So **`total == represented` is the honest "nothing was hidden" test**, and `complete` is not it. The
same wiki line tells callers the opposite - *"Gate on `diagnosticSummary`, never `diagnostics.length`;
shaping is not cleanliness, and `complete: false` is the decisive flag"* - which is exactly the gate
that misfires.

## Measured

`Content/FPS/Weapons/Meshes/SM_WPN_Pistol.pwmodel`, compiled with `diagnosticLimit: 20`:

```
"diagnosticSummary": { "total": 6, "errors": 0, "warnings": 6,
                       "emitted": 3, "represented": 6, "collapsedOccurrences": 3,
                       "suppressedBySeverity": 0, "suppressedByLimit": 0,
                       "complete": false, "limit": 20, "collapse": true }
```

Six warnings, six represented, nothing suppressed at any limit, and `complete: false`. The only way
to make it true is `collapse: false` - i.e. to turn off a shaping feature in order to satisfy a flag
that is documented as reporting whether shaping hid anything.

## Why it matters at all

It is Low because the numbers next to it are right and a caller who reads `suppressedByLimit` gets
the truth. It is still worth fixing because it costs the wiki's own recommended gate its meaning: an
agent told to gate on `complete` either wires the gate to fail on every real document or learns to
ignore the field, and the second outcome is how a genuine `suppressedByLimit` truncation gets missed
later.

**Fix:** compute it as `Represented == Diagnostics.Num()` (equivalently
`suppressedBySeverity + suppressedByLimit == 0`), which is what the wiki sentence already describes.
If the emission-count meaning is wanted as well, publish it under a second name rather than
overloading this one.

## History
- `#1-complete-false-with-nothing-suppressed` `OPEN` reporter - Hit while compiling a two-document weapon split. `model.compile` on `SM_WPN_Pistol.pwmodel` at `diagnosticLimit: 20` returned `total 6 / represented 6 / suppressedBySeverity 0 / suppressedByLimit 0` and `complete: false`, because `collapse` (default true) folded four occurrences of `PWMODEL_UNUNIONED_OVERLAP` into one entry and `ModelCompileHandler.cpp:567` compares the EMITTED count against the total. The sibling document `SM_WPN_Pistol_Slide.pwmodel`, which happened to have no repeated diagnostic, returned `complete: true` on the same call shape - so the flag tracks whether a document repeats itself, not whether anything was hidden. `Saved/PinWright/wiki/model.compile.md:37` publishes both the correct identity (`total == represented + suppressedBySeverity + suppressedByLimit`) and the advice to treat `complete: false` as "the decisive flag", which contradict each other. Fix: compare `represented` rather than `emitted`.
