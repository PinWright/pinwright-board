---
id: B-find-by-tag-matchtype-silent-fallback
title: "actor.find_by_tag silently falls back to exact matching on an unrecognised matchType instead of rejecting it"
status: IN-REVIEW
severity: Medium
category: bug
tags: [actor, find-by-tag, filter, silent-fallback, misleading-success, typo-tolerance, survey-spinoff]
encounters: 1
lastSeen: 2026-08-13T05:57:34Z
---

# `actor.find_by_tag` swallows an unrecognised `matchType`

`actor.find_by_tag` declares `matchType` as *"'exact' (default) — FName equality on each tag.
'contains' — case-insensitive substring on each tag string."*
(`Source/PinWright/Private/Handlers/Actor/QueryHandler.cpp:237`). The implementation tests
for exactly one value and treats **everything else** — including a typo, a wrong-vocabulary
token borrowed from a sibling verb, or a value from a future mode — as `exact`:

```cpp
FString MatchType = Ctx.GetString(TEXT("matchType")).ToLower();       // :254
...
if (MatchType == TEXT("contains")) {                                  // :268
    ... substring match ...
} else {
    bMatches = Actor->ActorHasTag(TagName);                           // :276  <- catches everything
}
```

So `matchType:"substring"`, `matchType:"conatins"`, or the `matchMode` vocabulary the rest of
the plugin now uses (`contains` / `prefix` / `exact` / `regex`, from
`Utils/NameMatchFilter.h`) all silently narrow to exact FName equality. The response is a
well-formed, plausible, **narrower-than-requested** set with no error, no warning, and no
echo of the mode that was actually used — the same silent-wrong-answer shape as the parent
ticket `B-actor-list-filter-case-mismatch`, and one that reads as "nothing is tagged that
way" rather than "your mode was ignored".

The dispatcher's strict `UNKNOWN_PARAMS` rejection does not help here: `matchType` **is** a
declared parameter, so only its *value* is unvalidated.

**Workaround:** pass `matchType:"contains"` character-exact, or omit it; verify a
substring-mode result by spot-checking that a known partial-tag match is present.

**Fix:** validate the value. Reject an unrecognised `matchType` with a typed error
(`INVALID_MODE`, already registered by `B-actor-list-filter-case-mismatch`'s fix) whose
message enumerates the accepted tokens — the same shape the board adopted for
`configure_slot_behavior`'s `availableBehaviorTypes` — and echo the resolved `matchType` on
success so a caller can prove which semantics produced the result. Better: adopt the shared
`NameMatch::FFilter` vocabulary (`matchMode` + `caseSensitive`, with `matchType` kept as a
back-compat alias mapping `exact`/`contains`) so this verb stops being a private
two-token policy, and its read-only twin `system.inspect.find_by_tag` gains the same modes.

severity rationale: impact=silent wrong data (a silently narrowed result set presented as a
complete answer, on a verb whose whole purpose is "find everything tagged X"), but the caller
must first supply a bad token, so it is not hit on a correct call x reach=tag audits are a
common orientation read -> Medium.

## Related

- `B-actor-list-filter-case-mismatch` — the parent survey ticket; this defect is item (e) of
  its `#3-survey-spinoffs` list and the detail stays recorded there. Its `#2` fix registered
  the `INVALID_MODE` error code and built the shared `NameMatch::FFilter` this should adopt;
  it deliberately did not extend to `actor.find_by_tag`.
- `F-inspect-list-actor-tags` — the tag-enumeration gap that makes callers guess tag names in
  the first place, and which notes `system.inspect.find_by_tag` lacks even the `contains`
  mode this verb has.
- Sibling spinoffs: `B-blueprint-references-casesensitive-noop`,
  `B-property-list-propertynames-case-mismatch`,
  `B-bulk-rename-docs-behaviour-mismatch`,
  `B-asset-list-class-filter-case-divergence`.

## History
- `#1-split-from-actor-list-survey` `OPEN` reporter — "Split out of `B-actor-list-filter-case-mismatch` `#3-survey-spinoffs` item (e), flagged there as out of scope and wanting its own ticket. Re-verified at HEAD: `QueryHandler.cpp:254` lowercases `matchType`, `:268` tests `== 'contains'`, and the `else` at `:276` runs `Actor->ActorHasTag(TagName)` — so every unrecognised value (typo, `substring`, or the plugin-wide `matchMode` tokens `prefix`/`regex`) silently becomes exact FName equality. No error, no warning, no echo of the mode used; the caller gets a silently narrowed set that reads as 'nothing is tagged that way'. The dispatcher's `UNKNOWN_PARAMS` guard does not catch it because `matchType` is a declared param and only its value is unvalidated. Fix: reject unknown values with the already-registered `INVALID_MODE` enumerating accepted tokens (matching the `configure_slot_behavior` `availableBehaviorTypes` precedent), echo the resolved `matchType` on success, and preferably adopt the shared `NameMatch::FFilter` vocabulary with `matchType` retained as a back-compat alias, extending the same modes to the read-only twin `system.inspect.find_by_tag`."

- `#2-unknown-matchtype-refused` `IN-REVIEW` developer — Fixed by rejecting the value instead of guessing at it. `Handlers/Actor/QueryHandler.cpp` now normalises `matchType` through `PinWright::NormalizeToken` (same trim/lowercase/`-`→`_` folding `NameMatch::Parse` uses), accepts `exact` (default) and `contains` (alias `substring`), and refuses anything else with `INVALID_MODE` naming both accepted tokens and quoting what was sent; the `else` branch that swallowed every unrecognised value is gone. The response now echoes the resolved `matchType` in its canonical spelling plus the queried `tag`, so a caller can prove which semantics produced the rows — without that echo the silent fallback was undetectable from the response. Deliberately NOT adopting `NameMatch::FFilter` here: its `caseSensitive` knob cannot be honoured against `FName` equality (`FName` compares case-insensitively), so accepting it would recreate this very defect one parameter over, and `prefix`/`regex` are not implemented for tags. `system.inspect.find_by_tag` is unchanged and still exact-only; extending it is a separate feature, not this bug. Tests (failure-direction, `Tests/Actor/TestActorFindByTagMatchType.cpp`): `PinWright.actor.find_by_tag.UnknownMatchTypeIsRefused` requires an INVALID_MODE refusal for `conatins` / `prefix` / `regex` / `substring_typo` — restoring the fallback turns each into a success — and `PinWright.actor.find_by_tag.ResolvedMatchTypeIsEchoed` requires the canonical echo for the omitted default, `exact`, `contains`, `substring` and `  Contains `; dropping the echo leaves it no field to read. Both compile clean (`-SingleFile -NoHotReloadFromIDE`, `Result: Succeeded`). Docs: new `### actor.find_by_tag` section in `Docs/wiki-src/actor.md`. Commits `2714ab94`, `758cf678`.
