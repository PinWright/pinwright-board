---
id: B-actor-list-filter-case-mismatch
title: "actor.list filter is case-insensitive but documented case-sensitive, silently inflating prefix counts"
status: DONE
severity: High
category: bug
tags: [actor, docs, filter, silent-wrong-data]
encounters: 1
lastSeen: 2026-08-13T02:18:00Z
---

# actor.list filter is case-insensitive but documented case-sensitive, silently inflating prefix counts

`actor.list`'s wiki page documents `filter` as "Substring filter applied to actor label and name
(**case-sensitive**)". The implementation matches case-**in**sensitively. Because `totalMatches`
reports the full untruncated match count, a caller using `filter` to count actors by naming-convention
prefix gets a silently wrong number with no signal that anything is off.

Observed on `/Game/Maps/Dota2_Blockout` (UE 5.8):

- `filter:"SH_"` → `totalMatches: 98`, first match `Brush0` / `Brush_0`. It matched the lowercase
  `sh_` inside "Bru**sh_**0". True count of labels actually starting with `SH_` is **4**.
- `filter:"OP_"` → `totalMatches: 15`, first match `RN_Power_Top_Pad`, matching the lowercase `op_`
  inside "T**op_**Pad". True count is **4**.

Both are ~4x and ~24x overstatements. The failure is quiet: the response looks perfectly well-formed,
and an agent doing crash-damage assessment or a before/after actor audit will report fabricated
numbers off the back of it. The only reason it was caught here was that the returned sample row was
eyeballed and obviously did not carry the requested prefix.

Also note `filter` is a substring match anywhere in label or name, not an anchored prefix match, so
even with the case issue resolved, `filter:"BR_"` cannot express "labels starting with `BR_`".

**Workaround:** do not use `filter` for counting. Pull the full list
(`actor.list {fields:["label","class"]}`, which spills to a `Saved/PinWright/HttpResponses/*.json`
file on any populated level) and count with an anchored regex client-side.

**Fix:** decide which behavior is intended and make code and docs agree.
- If case-insensitive is intended, correct the `filter` param help in `docs/wiki-src/actor.md`
  (the generated `Saved/PinWright/wiki/actor.list.md` inherits it) and say so explicitly, since
  `actor.find_by_name` already documents itself as case-insensitive and the mismatch between the two
  pages is what makes the current wording credible.
- Better: add an anchored/`prefix` option, or a `matchCase` flag, so convention-prefix counting —
  a common read on any level with a naming scheme — is expressible without a client-side pass.

## Split-out tickets (the `#3-survey-spinoffs` list)

The five survey defects recorded in `#3-survey-spinoffs` now each have their own OPEN ticket,
so the fix picker can see them. The detail below stays here as the record of where they were
found; the tickets carry the re-verified source citations.

| `#3` item | Ticket | Severity |
|---|---|---|
| (a) `blueprint.references` `caseSensitive` no-op | `B-blueprint-references-casesensitive-noop` | High |
| (b) `property.list` `propertyNames` case mismatch | `B-property-list-propertynames-case-mismatch` | Medium |
| (c) `asset.bulk_rename` x3 docs-vs-behaviour | `B-bulk-rename-docs-behaviour-mismatch` | High |
| (d) `asset.list` vs `asset.search` class-filter case | `B-asset-list-class-filter-case-divergence` | Medium |
| (e) `actor.find_by_tag` silent `matchType` fallback | `B-find-by-tag-matchtype-silent-fallback` | Medium |

(c) is one ticket rather than three: all three mismatches sit in one verb, one function and
one contiguous block (`AssetWorkflowHandler.cpp:321-396`) and share a single fix, matching
this board's per-verb/per-code-path grouping precedent. Each of the five re-verified against
HEAD source at filing time; all five cite `Utils/NameMatchFilter.h` (this ticket's `#2` fix)
as the vocabulary to converge on.

## History
- `#1-initial-repro` `OPEN` reporter — `actor.list {filter:"SH_", limit:1, namesOnly:true}` on
  `/Game/Maps/Dota2_Blockout` returns `totalMatches: 98` with first match `Brush_0`; true `SH_`-prefixed
  count is 4. Same for `filter:"OP_"` → 15 vs true 4 (matched `Top_Pad`). Docs for `filter` claim
  case-sensitive. Found while auditing actor counts after the crash in
  `B-python-execute-reentrant-gc-crash`, where it would have produced a fabricated damage report.
- `#2-shared-matcher-and-optin-modes` `IN-REVIEW` developer — **Confirmed semantics:** `QueryHandler.cpp:90`
  was `Label.Contains(Filter) || Name.Contains(Filter)`; `FString::Contains` defaults to
  `ESearchCase::IgnoreCase` (`UnrealString.h:1194`), so `filter` was a case-**IN**sensitive **substring**
  match against label OR internal name. Both surprises confirmed; the wiki claim of case-sensitivity was
  simply wrong.

  **Fix (source only — NOT yet compiled or runtime-verified):**
  - New shared matcher `Source/PinWright/Private/Utils/NameMatchFilter.{h,cpp}` — `NameMatch::FFilter`
    + `Parse` / `Require` / `AddFilterEcho` + the two `FParamSpec` builders, modelled on
    `Utils/ComponentReadFilter` and `Utils/AssetCreatePolicy`.
  - Two opt-in params, names/aliases/defaults identical across every consumer:
    `matchMode` (`match_mode`) = `contains` (alias `substring`) | `prefix` (alias `starts_with`) |
    `exact` | `regex`, default **`contains`**; `caseSensitive` (`case_sensitive`), default **`false`**.
    Vocabulary deliberately matches the already-shipped `blueprint.graph.find_nodes`.
  - **Back-compat:** defaults reproduce the old `Contains` + `IgnoreCase` behaviour exactly, so
    `filter:"SH_"` still matches `Brush_0` when neither param is passed. Only additive response fields.
  - Applied to `actor.list` (`Handlers/Actor/QueryHandler.cpp`) and, because their docs carried the same
    false "(case-sensitive)" claim, `system.inspect.list_objects` and
    `system.inspect.find_objects_by_class` (`Handlers/Environment/EnvironmentHandler.cpp`).
    A parallel workstream adopted the same helper for `spatial.raycast`'s `actorFilter` while this was in
    flight, so four verbs now share one vocabulary; only the pattern key differs per verb.
  - Typed errors: new `INVALID_PATTERN` (registered in `Handlers/ErrorCodes.h`) for a regex that does not
    compile — ICU swallows compile errors so an unvalidated bad pattern silently matches nothing;
    detection uses a `(?:PATTERN)?`-against-empty-input probe. `INVALID_MODE` for an unknown `matchMode`;
    `INVALID_ARGUMENT` for `matchMode`/`caseSensitive` with no `filter` (would return the whole level).
  - Responses now echo `filter` + `matchMode` + `caseSensitive` whenever filtering, so a counting caller
    can prove which semantics produced `totalMatches`. Unfiltered responses are byte-identical to before.
  - Also fixed in the same pass: `namesOnly`/`fields` documented `names_only`/`field` as accepted but did
    not DECLARE them, so the dispatcher rejected both with `UNKNOWN_PARAMS` — same docs-vs-behaviour class.
  - Docs corrected: `Docs/wiki-src/actor.md` (`### actor.list` rewritten with the `SH_`/`Brush_0` worked
    example), `Docs/wiki-src/system.inspect.md` (removed the false "(case-sensitive)"; added a
    `### system.inspect.find_objects_by_class` section), `Docs/wiki-src/level-blockout.md:179`
    ("Tags are case-sensitive" was wrong — `FName` tag matching is case-insensitive),
    `Docs/error-code-catalog.md`.
  - Tests: `Source/PinWright/Private/Tests/World/TestActorListFilterMatchMode.cpp` — 4 automation tests
    pinning the exact reported failure in both directions (legacy default STILL matches `Brush_0`;
    `caseSensitive`/`prefix`/anchored-regex do not), the typed rejections, the declared-param contract,
    and an end-to-end run against two spawned probe actors (`SH_ActorListFilterProbe` vs
    `Brush_ActorListFilterProbe`).

  **Not done / for the tester:** never compiled (a separate integration agent owns the build) and never
  run against a live editor. Runtime verification commands are in the handoff note; the decisive one is
  `actor.list {filter:"SH_", matchMode:"prefix", caseSensitive:true}` on `/Game/Maps/Dota2_Blockout`
  returning `totalMatches: 4` while the bare `{filter:"SH_"}` call still returns 98.
  Deliberately NOT extended to the other filter-taking verbs the survey flagged (`volume.get_volumes_info`,
  `skeleton.list_bones`, `gameplay_tags.list`, `asset.search`, `actor.find_by_tag`, …) — see
  `#3-survey-spinoffs` for the separate defects found there.
- `#3-survey-spinoffs` `IN-REVIEW` developer — a plugin-wide survey of every filter/pattern-taking verb
  turned up further docs-vs-behaviour defects that are **out of scope here and still unfixed**; they want
  their own tickets:
  (a) `blueprint.references` — `caseSensitive:true` is a **no-op** on the non-`exactTarget` path
  (`Handlers/Blueprint/BlueprintGraphHandler.cpp:120` calls `Contains` with no `ESearchCase`).
  (b) `property.list` — `propertyNames` is documented "(case-sensitive)" but is matched via
  `TSet<FString>::Contains`, which is case-INsensitive (`Handlers/Utility/UtilityPropertyHandler.cpp:1672`
  vs `:1748`); neither `nameMatch` nor `propertyNames` appears in `Docs/wiki-src/property.md`.
  (c) `asset.bulk_rename` — three mismatches: `searchText` documented case-sensitive but replaced with
  `ESearchCase::IgnoreCase`; prefix/suffix documented "only added if not already present" but applied
  unconditionally (double-prefixes on a re-run); summary claims prefix→suffix→replace but the code runs
  replace→prefix→suffix (`Handlers/Asset/AssetWorkflowHandler.cpp:322-396`).
  (d) `asset.list`'s class fallback uses `Equals` (case-SENSITIVE) while `asset.search`'s
  `classFilterMode:"exact"` uses `Equals(IgnoreCase)` — same conceptual filter, opposite case behaviour.
  (e) `actor.find_by_tag` silently falls back to `exact` on an unrecognised `matchType` instead of
  rejecting it, which is the same silent-wrong-answer shape as this ticket.
- `#4-runtime-verified-and-committed` `DONE` tester — Built (UE 5.8, `Result: Succeeded`, zero errors,
  confirmed by fresh+grown DLL timestamps, not exit code) and runtime-verified on
  `/Game/Maps/Dota2_Blockout`. **Back-compat:** `actor.list {filter:"SH_"}` still returns
  `totalMatches: 98` with `Brush0` first, now plus `matchMode:"contains", caseSensitive:false`.
  **Fix:** `{filter:"SH_", matchMode:"prefix", caseSensitive:true}` returns exactly **4**
  (`SH_Secret_R`, `SH_Secret_D`, `SH_Side_1`, `SH_Side_2`); `{filter:"OP_", matchMode:"prefix",
  caseSensitive:true}` returns **4** (was 15 via `T[op_]Pad`); `{filter:"^SH_", matchMode:"regex",
  caseSensitive:true}` returns 4, matching prefix mode. snake_case `match_mode`/`case_sensitive`/
  `names_only` all reach the handler and echo the canonical spelling. **Typed errors confirmed live:**
  `filter:"["`+`regex` → `INVALID_PATTERN`; `matchMode:"fuzzy"` → `INVALID_MODE`; `matchMode` with no
  `filter` → `INVALID_ARGUMENT`. Unfiltered `actor.list` emits none of the three echo keys.
  **system.inspect twins verified:** `list_objects {filter:"Landscape", matchMode:"prefix",
  caseSensitive:true}` → 1 match, and lowercase `"landscape"` → 0, so case sensitivity is genuinely
  enforced there too (they match internal object names, not labels, which is why `SH_` yields 0 —
  expected, not a defect). C++ suite 3588/3590 (the 2 failures are pre-existing
  `PinWright.localization.Validation.*`, unrelated); the 4 new
  `PinWright.actor.list.Filter*` tests all pass. Committed as `f18125f9`.
