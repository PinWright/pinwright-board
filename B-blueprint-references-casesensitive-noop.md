---
id: B-blueprint-references-casesensitive-noop
title: "blueprint.references accepts caseSensitive and silently ignores it — the substring match is always case-insensitive"
status: IN-REVIEW
severity: High
category: bug
tags: [blueprint, references, filter, case-sensitivity, silent-noop, accepted-and-ignored, survey-spinoff]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `blueprint.references`' `caseSensitive` is a no-op on the non-`exactTarget` path

`blueprint.references` declares a `caseSensitive` parameter, accepts it without complaint,
and then does not honour it on its default (substring) matching path. The result is worse
than the parameter being absent: the caller passes `caseSensitive:true`, gets no rejection
and no echo contradicting them, and reasonably believes the returned reference set is
case-exact when it is not.

## Source (verified 2026-08-13 at HEAD)

`Source/PinWright/Private/Handlers/Blueprint/BlueprintGraphHandler.cpp:109-120`:

```cpp
const FString TargetNormalized = bCaseSensitive ? TargetPath : TargetPath.ToLower();
...
const FString CandidateNormalized = bCaseSensitive ? Candidate : Candidate.ToLower();
return bExactTarget
    ? CandidateNormalized.Equals(TargetNormalized, ESearchCase::CaseSensitive)
    : CandidateNormalized.Contains(TargetNormalized);   // :120 — no ESearchCase
```

The `bCaseSensitive` flag only decides whether the two strings get lowercased first. On the
`bExactTarget` path that is sufficient, because the comparison then pins
`ESearchCase::CaseSensitive`. On the default `Contains` path it is not: `FString::Contains`
defaults to `ESearchCase::IgnoreCase` (`UnrealString.h:1194`), so leaving both strings in
their original case changes nothing — the match stays case-insensitive either way. The flag
is dead on the path callers actually use.

The same omission repeats in `MatchesNodeTypeFilter` a few lines below (`:~130-134`):

```cpp
const FString CandidateName = bCaseSensitive ? NodeTypeName : NodeTypeName.ToLower();
...
return CandidateName.Contains(NodeTypeNormalized) || CandidatePath.Contains(NodeTypeNormalized);
```

so the `nodeType` filter is unconditionally case-insensitive too, for the same reason.

**Workaround:** do not rely on `caseSensitive` here. Pull the unfiltered/loosely-filtered
reference set and re-filter client-side with a case-exact comparison.

**Fix:** pass the case mode into the comparison rather than pre-normalising —
`Candidate.Contains(Target, bCaseSensitive ? ESearchCase::CaseSensitive : ESearchCase::IgnoreCase)`
— on both `MatchesTargetFilter` and `MatchesNodeTypeFilter`, dropping the now-pointless
`ToLower()` pre-pass. Better: adopt the shared `NameMatch::FFilter`
(`Utils/NameMatchFilter.h`) that `B-actor-list-filter-case-mismatch`'s fix introduced and
`actor.list` / `system.inspect.list_objects` /
`system.inspect.find_objects_by_class` / `spatial.raycast` already speak, so
`blueprint.references` gets the same `filter` + `matchMode` + `caseSensitive` vocabulary
instead of a fifth private policy — and echo the resolved mode in the response so a caller
can prove which semantics produced the result set. **Note the ordering coupling:** that
helper landed in the same unbuilt working tree as the actor.list fix; this change must build
alongside it.

severity rationale: impact=silent wrong data on a normal path, with the aggravating factor
that the parameter is *accepted* (a caller has positive, false confirmation that the filter
is exact — worse than a `UNKNOWN_PARAMS` rejection would be) x reach=`blueprint.references`
is a common audit/refactor read -> High.

## Related

- `B-actor-list-filter-case-mismatch` — the parent survey ticket; this defect is item (a) of
  its `#3-survey-spinoffs` list and its detail stays recorded there. That ticket's fix built
  the shared `NameMatch::FFilter` this one should adopt.
- Sibling spinoffs split out of the same list:
  `B-property-list-propertynames-case-mismatch`,
  `B-bulk-rename-docs-behaviour-mismatch`,
  `B-asset-list-class-filter-case-divergence`,
  `B-find-by-tag-matchtype-silent-fallback`.

## History
- `#1-split-from-actor-list-survey` `OPEN` reporter — "Split out of `B-actor-list-filter-case-mismatch` `#3-survey-spinoffs` item (a), which flagged it as out of scope there and wanting its own ticket. `blueprint.references`' `caseSensitive` is accepted and silently ignored on the non-`exactTarget` path. Re-verified at HEAD: `BlueprintGraphHandler.cpp:109-120` uses `bCaseSensitive` only to decide whether to `ToLower()` both operands, then calls `CandidateNormalized.Contains(TargetNormalized)` with no `ESearchCase` (`:120`); `FString::Contains` defaults to `ESearchCase::IgnoreCase` (`UnrealString.h:1194`), so the match is case-insensitive whether the flag is set or not. The `bExactTarget` branch is unaffected because it pins `Equals(..., ESearchCase::CaseSensitive)`. Same omission in `MatchesNodeTypeFilter` (`:~130-134`), so the `nodeType` filter is also unconditionally case-insensitive. Accepted-and-ignored is worse than absent: the caller gets no rejection and believes the result set is case-exact. Fix: thread `ESearchCase` into both `Contains` calls, or better adopt the shared `NameMatch::FFilter` (`Utils/NameMatchFilter.h`) already used by actor.list / system.inspect.* / spatial.raycast and echo the resolved mode; note the helper is in the same unbuilt tree, so the changes must build together."
- `#2-additional-current-case-audit` `OPEN` reporter — Additional evidence: **Adversarial review A — keep; the defect remains in current source, with the title's target-path claim also applying to the nodeType filter.** Actuality: CONFIRMED CURRENT. Framing: the accepted `caseSensitive` option is a silent false-success on substring matching; `exactTarget` is correctly case-aware, so the ticket should retain that boundary rather than claim every matching mode is broken. Proposed fix: INCOMPLETE, explicit `ESearchCase` on both current `Contains` branches is the compatible core fix, but adopting `NameMatch::FFilter` verbatim would conflate this RPC's independent `targetPath` and `nodeType` patterns and its `exactTarget` switch; the response also still needs a resolved-mode echo or a rejection when a case modifier has no active filter. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:33-36` declares the filters and accepts `caseSensitive`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:54-55` reads `bExactTarget`/`bCaseSensitive`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:109-120` lowercases only for the insensitive path then calls `Contains` without a search-case argument; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:123-135` repeats the same omission for node type name/path; `C:\UE_5.8\Engine\Source\Runtime\Core\Public\Containers\UnrealString.h.inl:1191-1194` defines `FString::Contains`' default as `ESearchCase::IgnoreCase`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:292-300` echoes target/node filters but not `caseSensitive`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Blueprint\TestBlueprintHandlers.cpp:2651-2660` covers only missing-asset registration/validation, not matching semantics; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\NameMatchFilter.h:3-9,113-133` shows the shared helper's one-pattern contract and current consumer set. Runtime: NOT VERIFIED. Recommendation: KEEP; implement the explicit case mode while preserving `targetPath`/`nodeType`/`exactTarget`, add behavior tests for both filters and the echo/no-filter guard, then build and run a live Blueprint fixture verification.
- `#3-additional-case-filter-skeptic` `OPEN` reporter — Additional evidence: **Adversarial review B — keep; A's core defect finding survives, but the helper integration needs a two-filter design.** Actuality: CONFIRMED CURRENT. Framing: I agree that `exactTarget` honors the flag and the substring branches do not; no duplicate or later source fix was found. I disagree that a verbatim `NameMatch::Require` is sufficient: this handler has independent `targetPath` and `nodeType` patterns, while `NameMatch::Parse` is single-pattern; a naive guard could reject a valid nodeType-only query. Proposed fix: INCOMPLETE, explicit `ESearchCase` fixes matching but is a local band-aid unless shared matching is adapted for both filters and the response echoes `caseSensitive` when either is active. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:109-135`; `C:\UE_5.8\Engine\Source\Runtime\Core\Public\Containers\UnrealString.h.inl:1191-1194`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\NameMatchFilter.h:64-133`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Blueprint\BlueprintGraphHandler.cpp:292-300`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Blueprint\TestBlueprintHandlers.cpp:2651-2660`. Runtime: NOT VERIFIED. Recommendation: KEEP; extend/reuse the matcher without changing `exactTarget` scope, add case-direction and echo/no-filter tests for both patterns, then build and run a live Blueprint fixture.

- `#4-case-mode-threaded-through-shared-matcher` `IN-REVIEW` developer — Fixed in `Handlers/Blueprint/BlueprintGraphHandler.cpp`. Both filters now run through `NameMatch::FFilter` (`Utils/NameMatchFilter.h`), whose premise is that every comparison passes `ESearchCase` explicitly; the `ToLower()` pre-pass is gone. Answering reviews A and B: the struct is built BY HAND rather than through `NameMatch::Parse`, because Parse is single-pattern while this verb carries two independent patterns (`targetPath`, `nodeType`) under one shared case modifier plus its own `exactTarget` switch — so the wire shape is unchanged and a nodeType-only query is not rejected. `exactTarget` semantics are preserved exactly: lower-both-then-`Equals(CaseSensitive)` is the same relation as `Equals(IgnoreCase)`, which is what `EMode::Exact` with `bCaseSensitive=false` evaluates. Only the substring path changes, and only when the caller asks for case sensitivity. Also added, both requested by the reviews: `caseSensitive` supplied with neither filter, and `exactTarget` supplied with no `targetPath`, are refused with `INVALID_ARGUMENT` (a modifier with nothing to modify returns the whole unfiltered set while looking filtered), and the resolved `caseSensitive` is echoed whenever either filter is active. Tests (failure-direction, `Tests/Blueprint/TestBlueprintReferencesCaseSensitive.cpp`): `PinWright.blueprint.references.CaseSensitiveNarrowsTheSubstringMatch` builds a synthetic in-memory actor blueprint, matches the lowercase fragments `k2node` and `/script/blueprintgraph` against node class names/paths, and requires the case-sensitive count to be strictly LESS than the case-insensitive one on both filters — restoring the pre-normalisation makes them equal; each half emits the assertions-skipped marker rather than comparing 0 with 0. `PinWright.blueprint.references.CaseModifierIsEchoedOrRefused` pins the echo and the two INVALID_ARGUMENT guards. Both TUs compile clean (`-SingleFile -NoHotReloadFromIDE`, `Result: Succeeded`). Docs: `Docs/wiki-src/blueprint.md` now states which filters the flag governs, that the echo is how a caller proves which semantics ran, and why the `actor.list` `matchMode` vocabulary is deliberately not adopted here. Commits `bfe053a0`, `05825ccf`.
