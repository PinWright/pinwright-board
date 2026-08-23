---
id: B-asset-list-class-filter-case-divergence
title: "asset.list's class filter is case-sensitive while asset.search's classFilterMode:'exact' is case-insensitive — the same conceptual filter, opposite behaviour, neither documented"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset, filter, case-sensitivity, consistency, silent-empty-result, survey-spinoff]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `asset.list` and `asset.search` disagree on class-filter case

Two sibling verbs implement the same conceptual filter — "assets whose class is exactly X" —
with opposite case behaviour, and neither documents which it does.

`Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp:818` (`asset.list`, class
fallback path):

```cpp
if (!AssetClass.Equals(ClassFilter) && !AssetClassName.Equals(ClassFilter))
```

`FString::Equals` defaults to `ESearchCase::CaseSensitive`, so this comparison is
case-**sensitive**.

`AssetManageHandler.cpp:1127` (`asset.search`, `classFilterMode:"exact"` — the default mode):

```cpp
return Candidate.Equals(Filter, ESearchCase::IgnoreCase);
```

case-**in**sensitive, and its `contains` / `prefix` modes (`:1121`, `:1125`) likewise pin
`ESearchCase::IgnoreCase`.

The user-visible consequence is asymmetric and quiet: a `filter.class` value whose case does
not match the registry's spelling returns an **empty** `asset.list` result — no error, no
warning, just zero rows, which reads as "no such assets exist" rather than "your filter's
case was wrong". The same value works in `asset.search`. Since the two verbs already overlap
heavily (see `E-asset-search-vs-search-assets-overlap`), a caller moving between them for
pagination or field reasons silently changes matching semantics.

**Workaround:** use the registry's exact class spelling for `asset.list` (or use
`asset.search` with `classFilterMode:"exact"`, which is case-forgiving), and treat an empty
`asset.list` class-filtered result as inconclusive until confirmed with a case-insensitive
read.

**Fix:** converge the two on one policy and say so.
- Preferred: make `asset.list`'s fallback comparison `Equals(ClassFilter, ESearchCase::IgnoreCase)`
  to match `asset.search` and the rest of the plugin's filter vocabulary, which is
  case-insensitive by default (`NameMatch::FFilter`'s `caseSensitive` defaults to `false`).
- Better still: route both onto the shared `NameMatch::FFilter` (`Utils/NameMatchFilter.h`),
  so a caller gets the same `matchMode` + `caseSensitive` knobs on `asset.list` that
  `asset.search` approximates with `classFilterMode`, and both echo the resolved mode.
- Document the resulting semantics on both method sections in `docs/wiki-src/asset.md`;
  neither currently states any case behaviour.

severity rationale: impact=silent empty result on a normal path (the caller reads "nothing
matched" for a filter that is merely mis-cased) plus a cross-verb inconsistency that makes
the correct call unguessable x reach=`asset.list` / `asset.search` are common discovery
reads, but the mismatch only bites when the caller's class spelling differs in case from the
registry's -> Medium.

## Related

- `B-actor-list-filter-case-mismatch` — the parent survey ticket; this defect is item (d) of
  its `#3-survey-spinoffs` list and the detail stays recorded there. Its `#2` fix built the
  shared `NameMatch::FFilter` this should adopt.
- `E-asset-search-vs-search-assets-overlap` — the overlap between these two verbs, which is
  what makes the divergence reachable in practice.
- Sibling spinoffs: `B-blueprint-references-casesensitive-noop`,
  `B-property-list-propertynames-case-mismatch`,
  `B-bulk-rename-docs-behaviour-mismatch`,
  `B-find-by-tag-matchtype-silent-fallback`.

## History
- `#1-split-from-actor-list-survey` `OPEN` reporter — "Split out of `B-actor-list-filter-case-mismatch` `#3-survey-spinoffs` item (d), flagged there as out of scope and wanting its own ticket. Re-verified at HEAD: `asset.list`'s class fallback at `AssetManageHandler.cpp:818` uses `AssetClass.Equals(ClassFilter)` with no `ESearchCase`, and `FString::Equals` defaults to `ESearchCase::CaseSensitive`; `asset.search`'s `classFilterMode:'exact'` (the default) at `:1127` uses `Equals(Filter, ESearchCase::IgnoreCase)`, as do its `contains` (`:1121`) and `prefix` (`:1125`) modes. Same conceptual filter, opposite case behaviour, and neither method section in `docs/wiki-src/asset.md` states any case semantics. The failure is quiet: a mis-cased `filter.class` yields an empty `asset.list` result that reads as 'no such assets', while the same value works in `asset.search`. Fix: converge on case-insensitive (matching `asset.search` and the plugin-wide `NameMatch::FFilter` default), ideally by routing both verbs onto the shared helper so both echo the resolved mode, and document it on both."

- `#2-converged-on-case-insensitive` `IN-REVIEW` developer — Converged on **case-insensitive**, and `asset.list` is the side that moved (`Handlers/Asset/AssetManageHandler.cpp`, class post-filter). Three reasons that is the correct half rather than a coin toss: it is what `asset.search`'s `classFilterMode` already does in all three modes, what `NameMatch::FFilter` defaults to plugin-wide, and what `asset.list` ITSELF already did one step earlier — `ResolveUClass` resolves the class name case-insensitively when it builds the `FARFilter`, so the case-sensitive post-filter contradicted the verb's own front half. Both comparisons now pass `ESearchCase` explicitly instead of leaning on a default; leaning on the defaults is exactly how the two drifted apart, since `Equals` defaults `CaseSensitive` and `Contains` defaults `IgnoreCase`. `asset.list` now echoes `classFilter` / `classFilterMode:"exact"` / `classFilterCaseSensitive:false`, the same fields `asset.search` emits for the same concept. **Same file, same defect class, fixed alongside:** `asset.search`'s `classFilterMode` let every unrecognised value fall through to the exact branch, so `regex` or a typo ran a comparison the caller did not ask for while the echoed mode agreed with them; unknown values are now refused with `INVALID_MODE` naming the three accepted tokens, and `substring` / `starts_with` fold to their canonical spellings. Tests (failure-direction, `Tests/Assets/TestAssetClassFilterCaseParity.cpp`): `PinWright.asset.list.ClassFilterMatchesSearchOnCase` requires the lowercase and canonically-cased totals to be EQUAL (restoring the case-sensitive `Equals` collapses the lowercase call to zero), requires the echoed policy fields, and asserts the sibling verb accepts the same spelling so a future divergence fails there rather than in a caller's empty page; it emits the assertions-skipped marker if the mount holds nothing of the probe class. `PinWright.asset.search.UnknownClassFilterModeIsRefused` pins the refusal and the alias folding. Both TUs compile clean (`-SingleFile -NoHotReloadFromIDE`, `Result: Succeeded`). Docs: `Docs/wiki-src/asset.md` states the policy on BOTH method sections and corrects the `asset.list` paragraph that described the post-filter as a plain `Equals`. Commits `6e78c83c`, `5d7ac4fb`.
