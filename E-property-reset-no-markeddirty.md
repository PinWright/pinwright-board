---
id: E-property-reset-no-markeddirty
title: "property.reset honours markDirty but reports no markedDirty field, so the observed-state read-back that makes the guarantee checkable exists on property.set only"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [property, property-reset, mark-dirty, measured-vs-requested, response-honesty, readback-omits-field, verification]
encounters: 1
lastSeen: 2026-08-28T13:40:00+05:00
---

# The guarantee holds on `property.reset`. You just cannot see it from the response

`B-property-set-markdirty-false-still-dirties` fixed two things at once, and only one of
them reached `property.reset`:

- the **behaviour** — `Modify(bMarkDirty)` plus a package-scoped restore after
  `PostEditChange()` — landed on both verbs, and is verified live on both
  (that ticket's `#3`);
- the **report** — `markedDirty` stamped from `TargetPackage->IsDirty()` instead of echoing
  the request parameter — landed on `property.set` only.

`property.set` routes all five of its emit sites through the shared `FinalizeApplied`
lambda (`UtilityPropertyHandler.cpp:1065`), whose `:1079` writes
`markedDirty` from observed package state. `property.reset` never got one: it does the
identical dirty restore at `:1457`, then builds its payload by hand at `:1461-1467` —
`objectPath`, `propertyName`, `oldValue`, `defaultValue`, `defaultSource`, `wasOverridden`,
`isOverridden` — and stops. No `markedDirty`, and no `applied` either.

Measured, not read off the source: `property.reset {propertyName:"FadeInTime",
markDirty:false}` against a clean `/Game/PinWrightScratch` fixture returned exactly

```json
{"objectPath":"…","propertyName":"FadeInTime","oldValue":3.75,
 "defaultValue":0.20000000298023224,"defaultSource":"class_cdo",
 "wasOverridden":true,"isOverridden":false, …}
```

and the package did stay clean — confirmed by `editor.list_dirty_packages` and by
`EditorLoadingAndSavingUtils.get_dirty_content_packages()`, which agreed. The verb did the
right thing and said nothing about it.

## Why this is more than a missing field

The point of the `markedDirty` fix was that the field reports **observed state** rather than
echoing the request. That is not a cosmetic distinction — it is what makes the guarantee
*falsifiable from the response alone*. The proof that the fix held is a call that asked for
`markDirty:false` and answered `markedDirty:true` (on a package that was already dirty): a
contradiction between request and report that the old request-echo could not physically
produce. On `property.reset` that proof is unavailable, because there is nothing to
contradict.

So the caller's fallback is `editor.list_dirty_packages`, and it is a poor substitute on
three counts:

- **It is a second RPC**, not atomic with the write.
- **It is process-wide.** It returns every dirty package in the editor, so the caller must
  filter by package name — and must know the package name, which `property.reset` does
  report.
- **It is racy in a shared editor.** Between the reset and the list call another agent can
  dirty or clean the same package, so the answer is not necessarily about your call. The
  `property.set` field is read inside the handler, on the package the handler just touched.

## Not a documentation defect

`Docs/wiki-src/property.md:92-97` lists `property.reset`'s response fields as `oldValue`,
`defaultValue`, `wasOverridden`, `isOverridden` — `markedDirty` is correctly **absent**. The
docs are honest; the capability is missing. That is precisely why this is `ergonomic` and not
a bug: nothing false is reported, nothing is stale, and the `markDirty` contract itself is
kept. Filed so the asymmetry is a decision someone made rather than an oversight nobody
noticed.

## Fix

Small and behaviour-free. `FinalizeApplied` (`:1065`) is a lambda local to `property.set`'s
handler, capturing `TargetPackage`, `bPackageWasDirty` and `PropertyName`. Hoist the
`markedDirty` half into a file-local helper taking `(TSharedPtr<FJsonObject>&, UPackage*)`
and call it from `property.reset`'s payload build after the restore at `:1457`, so both verbs
report the same observed-state field from one implementation. Do **not** re-derive it from
`bMarkDirty` at the reset site — that reintroduces the request-echo the parent ticket removed.

**Test.** `Source/PinWright/Private/Tests/Utility/TestPropertyMarkDirtyRespected.cpp` already
has `property.reset.MarkDirtyFalseLeavesPackageClean`, which asserts on `UPackage::IsDirty()`
only. Extend it to assert the response field agrees with the observed package state, and add
the counterfactual (`property.reset` with `markDirty:true` → `markedDirty:true`) so a
hardcoded `false` cannot pass. Both need the file's real-on-disk `/Game` fixture, not a
transient one — `MarkPackageDirty()` early-returns under an `RF_Transient` outer, which makes
every such assertion vacuous.

## Decide together, do not file twice

`property.reset` also omits **`applied`**, which `property.set` reports. A caller today infers
success from the absence of an error plus `isOverridden:false`. That is the same
response-shape asymmetry between the two verbs and the same one-line fix site, so it should be
settled in the same change rather than become a third ticket — but it is a weaker case than
`markedDirty` (`isOverridden:false` is a genuine success signal; there is no substitute at all
for the dirty read-back), so it is recorded here rather than driving the severity.

## Severity

**Medium.** The rubric's Medium names this case almost verbatim: *"a readback omits a field
and forces a fallback."*

*Not High.* High is silent false-success or stale data the caller trusts. Nothing here is
false: the behaviour is correct and verified, the docs do not claim the field, and the verb
reports no wrong value. The caller is under-informed, not misinformed.

*Not Low.* Low is pure friction — "a response spill that only forces a `Read`". The fallback
is a second, process-wide, race-prone RPC that answers a slightly different question, and the
specific thing lost is the ability to verify a documented guarantee from the response. That is
a capability gap, not a formatting annoyance.

*Reach.* `property.reset` is the less-travelled half of a documented pair, but it is a normal
path with a documented `markDirty:false` "transient checks" workflow, not a rare edge, so no
downward bump.

## Related

- `B-property-set-markdirty-false-still-dirties` (**DONE**) — the parent. Its `#2` fix routed
  all five `property.set` emit sites through the shared observed-state finalizer and gave
  `property.reset` none; its `#3` live verification confirmed the *behaviour* holds on both
  verbs and recorded this gap explicitly so it would not be buried inside a closed ticket.
- `E-property-set-no-measured-render-state` (OPEN) — nearest neighbour, same family
  (measured-vs-requested response honesty) and same severity, but a different axis: that one
  is about the renderer picking a `property.set` up, this one is about the dirty flag after a
  `property.reset`.
- `B-property-set-saved-true-not-persisted` (IN-REVIEW) — the ticket that introduced
  `applied` + `markedDirty` on `property.set` in the first place, which is why the pair is
  asymmetric at all.

## History
- `#1-observed-during-parent-verification` `OPEN` reporter — Found while verifying `B-property-set-markdirty-false-still-dirties` live at plugin HEAD `b79ba53e` on a clean `/Game/PinWrightScratch` `USoundMix` fixture. `property.reset {propertyName:"FadeInTime", markDirty:false}` returned `objectPath` / `propertyName` / `oldValue 3.75` / `defaultValue 0.2` / `defaultSource "class_cdo"` / `wasOverridden true` / `isOverridden false` and **no `markedDirty`** (and no `applied`); the package genuinely stayed clean, confirmed by `editor.list_dirty_packages` and `EditorLoadingAndSavingUtils.get_dirty_content_packages()`, which agreed — so the behaviour is right and only the report is missing. Source-confirmed: `property.set` stamps `markedDirty` from `TargetPackage->IsDirty()` inside the shared `FinalizeApplied` lambda (`UtilityPropertyHandler.cpp:1065`, field written at `:1079`, restore at `:1073`), while `property.reset` performs the identical restore at `:1457` and then hand-builds its payload at `:1461-1467` with no equivalent. Docs are honest rather than wrong — `Docs/wiki-src/property.md:92-97` does not list `markedDirty` among `property.reset`'s response fields — which is why this is filed `ergonomic`. What is actually lost is verification: the parent fix's whole point was that `markedDirty` reports observed state rather than echoing the request, and the proof it held was a `markDirty:false` call answering `markedDirty:true`, a contradiction the old echo could not produce; on `property.reset` that check cannot be run from the response at all, leaving `editor.list_dirty_packages` — a second, process-wide, race-prone call in a shared editor — as the only route. Proposed fix is to hoist the `markedDirty` half of `FinalizeApplied` into a file-local helper and call it from the reset payload build, plus extending the existing `property.reset.MarkDirtyFalseLeavesPackageClean` test (which today asserts only `UPackage::IsDirty()`) to check the field against observed state with a `markDirty:true` counterfactual. Dedup: searched the board for `markedDirty` / `marked-dirty` / `property-reset` / `mark-dirty` — nearest are the parent ticket above, `E-property-set-no-measured-render-state` (same family, different axis: render state on `set`, not dirty flag on `reset`), and `B-property-set-saved-true-not-persisted` (introduced the field on `set`); none covers this.
- `#2-hoisted-observed-state-stamp-into-shared-helper` `IN-REVIEW` developer — Hoisted the observed-state half of `property.set`'s `FinalizeApplied` into a file-local `StampMarkedDirty(TSharedPtr<FJsonObject>&, const UPackage*, bool)` in `Handlers/Utility/UtilityPropertyHandler.cpp` (defined above the `namespace {` helper block), and called it from both verbs: `FinalizeApplied` now delegates instead of stamping inline, and `property.reset`'s hand-built payload calls it after the dirty decision, so the field can only ever come from `TargetPackage->IsDirty()` and the two verbs cannot drift. The `bRequestedMarkDirty` third argument preserves `FinalizeApplied`'s existing no-package fallback verbatim (defensive only — `GetOutermost()` resolves to the transient package at worst); it is never consulted when a package exists, so no request-echo is reintroduced. Also settled `applied` in the same change as the ticket directed: `property.reset` now reports `applied:true` alongside `markedDirty`, matching `property.set`. **`property.reset` genuinely dirties** — `RootObject->Modify(bMarkDirty)` plus `MarkPackageDirty()` on the `markDirty:true` path, with the clean-package restore on the `false` path — so the field reports a real fact, not a fabricated `true`. **The `container.*` mutators do NOT share this gap and were deliberately not closed with it:** all 11 (`array.append/remove/clear/insert/set`, `map.set/remove/clear`, `set.add/remove/clear`) call a bare `RootObject->Modify()` with no `markDirty` param, no dirty baseline and no restore — grep confirms `RPC_PARAM_OPT("markDirty", …)` appears exactly twice in the file, on `property.set` and `property.reset` only. They are consistently always-dirty and violate no contract (the parent ticket recorded the same finding), so a `markedDirty` there would be a field with nothing to contradict, across 11 sites — a different change, not this one-liner. Docs: `Docs/wiki-src/property.md` now lists `applied` and `markedDirty` under `property.reset`'s response fields with the observed-state caveat and the mark-dirty-only note. Tests, in `Tests/Utility/TestPropertyMarkDirtyRespected.cpp` (real saved `/Game` `USoundMix` fixture, as the ticket required — a transient outer makes every dirty assertion vacuous): new shared `TestMarkedDirtyMatchesObservedState` helper asserts the field is PRESENT and EQUALS the state the test observed for itself, seeded to the wrong answer so a failed `TryGetBoolField` cannot pass by accident; extended `property.reset.MarkDirtyFalseLeavesPackageClean` with it plus `applied` and a no-`saved:true` check; added `property.reset.MarkDirtyTrueDirtiesPackage` (the dirty direction — blocks a hardcoded `false`) and `property.reset.MarkDirtyFalsePreservesPreexistingDirt` (the discriminating case: request says false, honest answer is true — the contradiction a request-echo cannot produce). All three fail pre-fix on the presence assertion alone, since `property.reset` emitted no `markedDirty` at all. Suite delta +2 tests; no id is a dot-prefix of another. Not compiled or run here per task constraint — needs a build + `PinWright.property.*` pass before DONE.
