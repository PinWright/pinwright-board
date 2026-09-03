---
id: B-property-list-propertynames-case-mismatch
title: "property.list documents propertyNames as case-sensitive but matches via TSet<FString>::Contains, which is case-insensitive; neither propertyNames nor nameMatch is documented in the property wiki overlay"
status: IN-REVIEW
severity: Medium
category: bug
tags: [property, filter, case-sensitivity, docs, silent-wrong-data, survey-spinoff]
encounters: 1
lastSeen: 2026-08-14T00:00:00Z
---

# `property.list`'s `propertyNames` allow-list is case-insensitive despite documenting the opposite

`property.list` declares `propertyNames` as *"Exact UPROPERTY FName allow-list
(case-sensitive); empty/missing = no filter"*
(`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:1672`). The filter is
applied at `:1749` as:

```cpp
if (PropertyNameFilter.Num() > 0 && !PropertyNameFilter.Contains(PropName))
```

`PropertyNameFilter` is a `TSet<FString>`, and `FString`'s `operator==` / `GetTypeHash` are
both case-**in**sensitive in UE — so `TSet<FString>::Contains` matches case-insensitively.
A caller who passes `propertyNames:["bHidden"]` also matches `BHIDDEN`, and, more
practically, a caller relying on the documented case-exactness to disambiguate between
similarly-named UPROPERTYs on a class with a naming convention gets a silently broader
result set than the contract promises.

Same docs-vs-behaviour class as `B-actor-list-filter-case-mismatch` (where `filter` was
documented case-sensitive and matched case-insensitively), and it is the same wrong direction:
the docs claim the *stricter* semantics, so the surprise is always a silent over-match.

Secondary gap flagged in the same survey: neither `nameMatch` nor `propertyNames` appears in
`docs/wiki-src/property.md`. They exist only as `RPC_PARAM_OPT` descriptions in the generated
page, which is where the false "(case-sensitive)" claim lives — so correcting the overlay
without correcting the param description leaves the lie in the generated wiki.

**Workaround:** treat `propertyNames` as case-insensitive; if case matters, post-filter the
returned `name` fields client-side.

**Fix:** decide the intended semantics and make code, param description, and overlay agree.
- If case-insensitive is intended (it matches `nameMatch`, which is *correctly* documented
  case-insensitive and correctly implemented with `ESearchCase::IgnoreCase` at `:1745`),
  simply correct the `propertyNames` description at `:1672` and say so.
- If case-sensitive is intended, switch the container to a case-sensitive comparison
  (e.g. `TSet<FString, FLocKeySetFuncs>`-style case-sensitive key funcs, or a linear
  `Equals(..., ESearchCase::CaseSensitive)` scan — the allow-list is small).
- Either way, add `nameMatch` / `propertyNames` to `docs/wiki-src/property.md`, and prefer
  routing this verb onto the shared `NameMatch::FFilter` vocabulary
  (`Utils/NameMatchFilter.h`) so `property.list` speaks the same
  `matchMode` + `caseSensitive` language as `actor.list` and friends.

severity rationale: impact=silent wrong data (an over-broad result set against a documented
exact contract), but the effect is additive rows a caller can still inspect rather than a
fabricated count x reach=`property.list` is a common read, and the allow-list is normally
used with distinct names where case collisions do not arise -> Medium.

## Related

- `B-actor-list-filter-case-mismatch` — the parent survey ticket; this defect is item (b) of
  its `#3-survey-spinoffs` list and its detail stays recorded there.
- Sibling spinoffs: `B-blueprint-references-casesensitive-noop`,
  `B-bulk-rename-docs-behaviour-mismatch`,
  `B-asset-list-class-filter-case-divergence`,
  `B-find-by-tag-matchtype-silent-fallback`.
- `F-property-list-name-filter` (DONE) — the ticket that added `nameMatch` / `propertyNames`;
  its `#3` describes `propertyNames` as an "exact-name allow-list", which is where the
  case-sensitivity claim originates.

## History
- `#1-split-from-actor-list-survey` `OPEN` reporter — "Split out of `B-actor-list-filter-case-mismatch` `#3-survey-spinoffs` item (b), flagged there as out of scope and wanting its own ticket. Re-verified at HEAD: `UtilityPropertyHandler.cpp:1672` documents `propertyNames` as an 'Exact UPROPERTY FName allow-list (case-sensitive)', but `:1749` filters with `PropertyNameFilter.Contains(PropName)` on a `TSet<FString>`, and FString's `operator==`/`GetTypeHash` are case-INsensitive in UE, so the allow-list matches case-insensitively. Same docs-claim-stricter-than-code shape as the parent ticket, so the failure is always a silent over-match. The sibling `nameMatch` filter at `:1745` is correct (`Contains(..., ESearchCase::IgnoreCase)` and documented as case-insensitive). Also confirmed: neither param appears in `docs/wiki-src/property.md`, so the only documentation is the param description that carries the false claim. Fix: pick the intended semantics, correct `:1672` (or make the set case-sensitive), document both params in the overlay, and prefer adopting the shared `NameMatch::FFilter` vocabulary."
- `#2-docs-corrected-to-match-code` `IN-REVIEW` developer — **Decision: docs changed to match code; behaviour deliberately left alone.** Rationale for NOT making the set case-sensitive: (1) `propertyNames` is an FName allow-list and UE's FName lookup is natively case-insensitive, so the code already behaves the way the plugin's other name lookups do; (2) UPROPERTY names are unique per class regardless of case, so unlike the parent ticket's unanchored substring `filter` this insensitivity **cannot** inflate a count or pull in an unrelated field — the worst case is `bhidden` resolving `bHidden`, which is helpful, not wrong. The parent's silent-over-match harm does not transfer, so the severity driver is absent; (3) the recommended `NameMatch::FFilter` remedy does **not** structurally apply — `Utils/NameMatchFilter.h` carries exactly ONE pattern plus `matchMode`/`caseSensitive`, whereas `propertyNames` is a set-membership test over N names. Adopting it would redefine the param, not fix it. So no `matchMode`/`caseSensitive` knobs were added to this verb, and the `#1` recommendation to route it onto the shared vocabulary is explicitly declined.

  **Changes (source + docs only — NOT compiled, NOT runtime-verified; the build is owned by another agent and this pass was barred from building):**
  - `Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:1672` — param help rewritten: the false "(case-sensitive)" is gone, replaced with "matched case-INsensitively (FName lookup semantics), so 'bhidden' selects 'bHidden'", plus "Whole-name equality, not substring - use nameMatch for substring" to head off the other likely misread. This string is what the generated wiki renders, so it was the load-bearing fix.
  - `Docs/wiki-src/property.md` — closed the secondary gap: added a "**The two name filters — both case-INsensitive**" block under `### property.list`, documenting `nameMatch` and `propertyNames` for the first time, naming the `TSet<FString>` mechanism, and recording that neither takes `matchMode`/`caseSensitive` and why.
  - **Bonus defect found and fixed in the same block:** `Docs/wiki-src/property.md:87` told users to "narrow with the `filter` param". `property.list` declares no `filter` param at all, so following that instruction returns `[UNKNOWN_PARAMS]`. Same docs-vs-behaviour family, and the doc was the wrong half. Corrected to point at `nameMatch`/`propertyNames` and to state the rejection explicitly.

  **Test impact: none.** Swept `Tests/Infra/` — no doc-contract test pins any case-sensitivity wording for this verb. The nearby pinned literals are `"nameMatch"` (`Tests/Infra/WikiDocTestHelpers.h:93`, asserted against `Docs/wiki-src/container.md`, untouched here) and `"actor.get_components"` / `"Component0"` (`Tests/Infra/TestWikiHandler.cpp:393-398`, both still present). Nothing was reworded around them, so nothing needed retargeting.

  **Not done / for the tester:** never compiled, never run. No behaviour changed, so the only build risk is the two edited string literals. Verification on the next build: `property.list {objectPath:<any actor>, propertyNames:["bhidden"]}` should return the `bHidden` row (proving insensitivity), and `property.list {objectPath:<any>, filter:"x"}` should return `UNKNOWN_PARAMS` (proving the old doc line was wrong).

  **Adjacent drift noticed and deliberately NOT touched** (different defect class, wants its own ticket): `Docs/wiki-src/property.md`'s `property.list` optional-param list still says `includeAll` defaults `false`, while `UtilityPropertyHandler.cpp:1662` calls it a "Deprecated alias ... (now default true; kept for back-compat)", and that same doc list omits the declared `editableOnly` param entirely.
