---
id: B-bulk-rename-docs-behaviour-mismatch
title: "asset.bulk_rename contradicts its own docs three ways: searchText is case-insensitive not case-sensitive, prefix/suffix are applied unconditionally (double-prefix on re-run), and the operation order is replace->prefix->suffix not prefix->suffix->replace"
status: IN-REVIEW
severity: High
category: bug
tags: [asset, bulk-rename, docs, case-sensitivity, idempotency, silent-wrong-data, survey-spinoff]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `asset.bulk_rename` disagrees with its documentation in three independent ways

All three defects live in one verb, one function, and one contiguous block
(`Source/PinWright/Private/Handlers/Asset/AssetWorkflowHandler.cpp:321-396`), and all three
are fixed by the same edit, so they are tracked as one ticket rather than three. Each is
independently observable and independently damaging; a partial fix leaves a live trap.

Bulk rename is a **destructive, redirector-creating** operation over an arbitrary list of
assets. Every one of these mismatches produces wrong asset names that then need a second
bulk rename to undo, and each undo leaves more redirectors.

## The three mismatches (source-verified 2026-08-13 at HEAD)

**1. `searchText` is documented case-sensitive; the replace is case-insensitive.**
Param description (`:328`): *"Substring to find in each leaf name; case-sensitive."*
Implementation (`:383`):

```cpp
NewName = NewName.Replace(*SearchText, *ReplaceText, ESearchCase::IgnoreCase);
```

So `searchText:"SM_"` also rewrites `sm_` anywhere in the leaf name. This is exactly the
`B-actor-list-filter-case-mismatch` shape (docs claim the stricter semantics) but on a
**mutating** verb, so the over-match renames assets the caller never intended to touch.

**2. `prefix` / `suffix` are documented conditional; they are applied unconditionally.**
Param descriptions (`:326-327`): *"only added if not already present"*.
Implementation (`:386-393`):

```cpp
if (!Prefix.IsEmpty()) { NewName = Prefix + NewName; }
if (!Suffix.IsEmpty()) { NewName = NewName + Suffix; }
```

No `StartsWith` / `EndsWith` guard. Re-running the same call — the natural response to a
partial failure, and the exact retry the transport's response-only timeout makes likely
(see the house rule that mutating handlers must be idempotent under client retry) — yields
`SM_SM_Rock`, then `SM_SM_SM_Rock`. The documented behaviour is precisely the idempotency
the code does not have.

**3. The documented operation order is the reverse of the executed order.**
Registered summary (`:321`): *"Operations are applied in order: prefix, suffix, then
search-replace."*
Implementation order (`:381-393`): **search-replace first**, then prefix, then suffix.

This is not cosmetic: it changes results whenever the prefix/suffix text intersects
`searchText`. Documented order applied to `Rock` with `prefix:"SM_"`, `searchText:"SM_"`,
`replaceText:"S_"` yields `S_Rock`; the real order yields `SM_Rock`. A caller reasoning from
the summary computes the wrong target names and cannot tell from the response, which reports
only the renames it performed.

**Workaround:** pass exactly one transformation per call (prefix only, or suffix only, or
search-replace only), never combine them; assume `searchText` is case-insensitive and choose
a substring with no case-variant collisions; and never re-run a prefix/suffix call — check
the current names first, because a retry double-applies.

**Fix:** make the code match the documented contract (the documented contract is the
better one on all three points), then re-state it:
1. `Replace(..., ESearchCase::CaseSensitive)`, or expose an explicit `caseSensitive` flag
   defaulting to the documented behaviour and echo the resolved mode.
2. Guard prefix/suffix with `StartsWith`/`EndsWith` so the verb is idempotent under retry.
3. Apply prefix -> suffix -> search-replace as documented, or correct the summary — but
   whichever way it lands, echo the applied order and the per-asset
   `{oldName, newName}` pairs so the caller can verify without a follow-up `asset.list`.
Because this changes rename output, treat it as behaviour-changing and consider a `dryRun`
sibling verb (`asset.preview_bulk_rename`) per the house split-read-from-mutate rule, so
callers can see the computed names before redirectors exist.

severity rationale: impact=silent wrong data on a **mutating, hard-to-undo** path (wrong
asset names + redirectors, with a documented-idempotent operation that double-applies on
retry) x reach=bulk rename is the standard naming-convention cleanup verb -> High.

## Related

- `B-actor-list-filter-case-mismatch` — the parent survey ticket; these three are item (c) of
  its `#3-survey-spinoffs` list and the detail stays recorded there. Its `#2` fix also
  established the shared `NameMatch::FFilter` case vocabulary
  (`matchMode` / `caseSensitive`, `Utils/NameMatchFilter.h`) that a `searchText` case flag
  here should reuse rather than reinvent.
- Sibling spinoffs: `B-blueprint-references-casesensitive-noop`,
  `B-property-list-propertynames-case-mismatch`,
  `B-asset-list-class-filter-case-divergence`,
  `B-find-by-tag-matchtype-silent-fallback`.

## History
- `#1-split-from-actor-list-survey` `OPEN` reporter — "Split out of `B-actor-list-filter-case-mismatch` `#3-survey-spinoffs` item (c), flagged there as out of scope and wanting its own ticket. Grouped as one ticket rather than three because all three mismatches are in one verb, one function and one contiguous block (`AssetWorkflowHandler.cpp:321-396`) and share a single fix — matching the board's per-verb/per-fix grouping precedent (`B-viewport-screenshot-writes-jpeg` split by code path, not by symptom). All three re-verified at HEAD: (1) `searchText` documented 'case-sensitive' at `:328` but replaced with `ESearchCase::IgnoreCase` at `:383`, so an over-match renames unintended assets on a MUTATING verb; (2) `prefix`/`suffix` documented 'only added if not already present' at `:326-327` but applied unconditionally at `:386-393` with no StartsWith/EndsWith guard, so a re-run — the natural retry after a partial failure, and one the response-only transport timeout invites — produces `SM_SM_Rock`; (3) the registered summary at `:321` claims 'prefix, suffix, then search-replace' while the code runs replace (`:381-383`) then prefix then suffix, which changes results whenever prefix/suffix text intersects `searchText` (`Rock` + prefix `SM_` + replace `SM_`->`S_` gives `S_Rock` documented vs `SM_Rock` actual). Fix: make the code match the documented contract on all three points (case-sensitive replace or an explicit echoed flag reusing the shared `NameMatch::FFilter` vocabulary; StartsWith/EndsWith guards for idempotency; documented operation order), echo the applied order and per-asset old/new name pairs, and consider a dryRun preview sibling since renames create redirectors."
- `#2-docs-corrected-first-behaviour-deferred` `IN-REVIEW` developer — **Decision: docs changed to match code, for now. `#1`'s recommendation (change the code to match the docs) is still the right END state and is NOT withdrawn — it is sequenced second, not rejected.**

  **Why docs-first rather than code-first, despite the docs being the better contract.** This is a mutating, redirector-creating verb over an arbitrary asset list, and this pass was explicitly barred from building or running the editor, so any behaviour change would ship unbuilt and unverified. The failure mode of a wrong behaviour change here is renaming the wrong assets and needing a second bulk rename to undo, each undo leaving more redirectors — strictly worse than the status quo. Meanwhile the docs were causing live harm every day they stayed wrong: a caller following the stated contract computes wrong target names, and the "only added if not already present" claim actively invites the retry that produces `SM_SM_Rock`. Correcting the docs removes that harm immediately at zero build risk. This also follows the parent ticket's own shape: `B-actor-list-filter-case-mismatch` made the honest description land first and kept existing behaviour as the default.

  **Changes (source strings + docs only — NOT compiled, NOT runtime-verified):**
  - `Handlers/Asset/AssetWorkflowHandler.cpp` registered summary — now states the real order (search-replace FIRST, then prefix, then suffix) and warns "NOT idempotent ... re-running the same call double-prefixes (SM_SM_Rock)".
  - `AssetWorkflowHandler.cpp` `prefix` / `suffix` param help — the false "only added if not already present" is gone from both, replaced with "Applied UNCONDITIONALLY - there is no already-present check", each naming its position in the real order.
  - `AssetWorkflowHandler.cpp` `searchText` param help — "case-sensitive" replaced with "matched case-INsensitively (ESearchCase::IgnoreCase), so 'sm_' also hits 'SM_'", plus "Applied BEFORE prefix/suffix".
  - `Docs/wiki-src/asset.md` — **the verb had no wiki section at all**; every word of its documentation was the param help, so the overlay could not have corrected it. Added a `### asset.bulk_rename` section covering all three mismatches with the worked `Rock`/`S_Rock`/`SM_Rock` order example, the redirector follow-up, and a safe one-asset-first recipe. Also recorded that it deliberately does NOT take the shared `matchMode`/`caseSensitive` pair, because it performs a replacement rather than a match — so `Utils/NameMatchFilter.h` supplies vocabulary here at best, not machinery. `#1`'s "reuse the shared NameMatch::FFilter vocabulary" should be read as: reuse the `caseSensitive` param NAME and default, not the `FFilter` type.

  **Test impact: none.** The only test touching this verb is `Tests/Assets/TestAssetHandlers.cpp:665-675`, a registration-existence smoke test asserting no literals. No `Tests/Infra/` doc-contract test references `asset.bulk_rename`, `searchText`, or the wording changed here, so nothing needed retargeting.

  **Line-number drift correction for future readers:** `#1` cites `:321-396`, `:326-328`, `:383`, `:386-393`. At HEAD before this edit those were `:324` (summary), `:327-329` (prefix/suffix/searchText), `:388` (`Replace(...IgnoreCase)`), `:391-398` (prefix/suffix application). The code was unchanged; only line numbers had shifted.

  **Still OPEN and deliberately not done — the behaviour half.** Nothing in the code path changed; all three defects remain live and the ticket must not be closed on the strength of this entry. Remaining work, for whoever holds the build: (1) `StartsWith`/`EndsWith` guards for prefix/suffix idempotency; (2) an explicit `caseSensitive` param (name and `false` default per `Utils/NameMatchFilter.h`) with the resolved value echoed; (3) settle the operation order deliberately and echo it; (4) echo per-asset `{oldName,newName}`; (5) the `asset.preview_bulk_rename` dryRun sibling. When those land, the docs added here must be revised in the same commit — they currently describe the defects as current behaviour, so a behaviour fix makes THIS page the stale one.
