---
id: B-asset-search-array-class-filter-silently-dropped
title: "asset.search accepts an array-valued classFilter, reads it as the empty string, applies no class filter at all and still echoes classFilterMode — the correctly-spelled key fails silently while a misspelling is refused, because the dispatcher validates parameter NAMES and never parameter TYPES"
status: OPEN
severity: High
category: bug
tags: [asset, asset-search, classFilter, classFilterMode, dispatcher, type-validation, silent-drop, silent-wrong-data, unknown-params, param-shape-drift]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Spell the key wrong and you are told. Spell it right and get 38 wrong answers.

`asset.search` with `classFilter: ["StaticMesh"]` returned **38 results, none of them a
StaticMesh** — materials, a SoundWave, actors — with `classFilterMode: "exact"` echoed in the
response.

The misspelling `classNames` is **correctly refused**. `ValidateHandlerParams`
(`Private/Dispatch/RpcDispatcher.cpp:91`) builds the declared-name set at `:128-132`, tests every
incoming JSON key against it at `:137`, and sends `UNKNOWN_PARAMS` at `:159` before the handler
runs. A caller who types the wrong key gets a clear error naming the valid parameters. A caller who
types the right key in the wrong shape gets a success payload, a full result set, and an echoed
mode field that says the filter was applied.

That asymmetry is the defect. It is not vegetation-specific and not really `asset.search`-specific:
it is a general asset-verb defect with a dispatcher-level root, and `asset.search` is the instance
that was measured.

## Read, or read and dropped? Both — and which one depends on the JSON type

This matters because the brief's two candidate diagnoses are each half right, and a fixer told the
wrong half will patch the wrong file.

**For a `string` value, `classFilter` is read AND applied.** It is not dead code.
`Private/Handlers/Asset/AssetManageHandler.cpp` (registration `:1332`):

```cpp
1354:    FString ClassFilter = Ctx.GetString(TEXT("classFilter"));
...
1511:        if (!ClassPathFilter.IsEmpty())          // classPathFilter wins
1518:        else if (!ClassFilter.IsEmpty())
1520:            if (!MatchesClassByMode(AssetClassName, ClassFilter) &&
1521:                !MatchesClassByMode(AssetClassPath, ClassFilter))
1523:                continue;
```

`MatchesClassByMode` is a handler-local lambda (`:1465-1482`) doing a manual post-filter over the
asset-registry results (loop head `:1507`). `classFilter` never reaches an `FARFilter`; the only
registry fields populated anywhere in the handler are `ClassPaths` / `bRecursiveClasses` (`:1416-
1417`, the `parentClassPath` branch) and `PackagePaths` / `bRecursivePaths` (`:1420-1421`,
`:1455-1456`). `ClassNames` is never used. So `classFilter: "StaticMesh"` works.

**For an `array` value it is read as `""` and the filter never runs.** The declared type is
`"string"` (`:1337`) and nothing enforces it:

- `Ctx.GetString` (`Private/Handlers/HandlerContext.cpp:17-24`) is `Payload->HasField(Key)` then
  `Payload->GetStringField(Key)`.
- `FJsonObject::GetStringField`
  (`C:/UE_5.8/Engine/Source/Runtime/Json/Private/Dom/JsonObject.cpp:474-477`) is
  `GetField<EJson::None>(FieldName)->AsString()` — `EJson::None` means *any* type is accepted.
- `FJsonValue::AsString` (`.../Private/Dom/JsonValue.cpp:26-36`) calls `TryGetString`, and on
  failure calls `ErrorMessage(TEXT("String"))` (`:414`) — a `LogJson` log line the handler cannot
  see — and **returns the empty string**.

So `ClassFilter == ""`, the `else if` at `:1518` is false, and every asset the registry returned is
kept. The response is then internally inconsistent in a way that hides the drop:

```cpp
1581:    if (!ClassFilter.IsEmpty())
1583:        Result->SetStringField(TEXT("classFilter"), ClassFilter);   // gated -> NOT emitted
...
1593:    Result->SetStringField(TEXT("classFilterMode"), ClassFilterMode);  // ungated -> emitted
```

`classFilter` is silently omitted from the echo while `classFilterMode` is published
unconditionally. The measured response matches exactly: it carried `classFilterMode: "exact"` and
no `classFilter`. A caller reading a mode field for a filter that is not in the response has no
reason to notice the filter is missing rather than assumed.

## The root is in the dispatcher, and it is one line wide

`FParamSpec::Type` is declared on every parameter and **never checked**. `grep -n "Spec.Type"
RpcDispatcher.cpp` returns three hits: `:52` and `:71` are `Spec.TypedAliases` (a different field),
and `:107` is the *missing-required-parameter* message, which prints the type as prose. There is no
`INVALID_PARAM_TYPE` path anywhere in the file. The gate at `:137` is `KnownParams.Contains(key)` —
name-only, by construction.

Every `RPC_PARAM_*` type string in the plugin is therefore documentation, not a contract. That is
what produces the asymmetry in this ticket's title: the name half of the declaration is enforced at
`:159`, the type half is not enforced at all, and the failure modes of the two halves could not be
further apart — a loud error versus a silent wrong answer.

## Why callers send an array here, which is what makes this a normal path

The sibling verb `blueprint.build_api_index` declares **the same parameter name as an array**
(`Private/Handlers/Blueprint/BlueprintApiIndexHandler.cpp:32`):

```cpp
32:        RPC_PARAM_OPT("classFilter", "array", "Limit scan to specific class names; omit to scan all classes")
38:    if (const TArray<TSharedPtr<FJsonValue>>* FilterArray = Ctx.GetArray(TEXT("classFilter")))
```

One name, two shapes, two namespaces, no signal at the boundary. The board already records callers
reaching the array shape twice, both times as an incidental step in an unrelated task:

- `B-add-montage-notify-time-dropped:26` — `asset.search { query: "Skeleton", classFilter:
  ["Skeleton"], limit: 5 }`
- `E-replace-node-noncallable-no-hint:68` — `{ query: "SetLight", classFilter: ["LightComponent"] }`

Neither ticket is about this; both simply used `asset.search` to find something, and neither author
had any reason to know their filter did nothing. That is the reach argument in concrete form: this
does not require an unusual call, it requires a caller who learned `classFilter` from the other
verb, or who guessed that a filter takes a list.

## Fix

Fix it in the dispatcher, not in `asset.search`. `ValidateHandlerParams`
(`RpcDispatcher.cpp:91-160`) already walks the declared specs (`:129-132`) and the incoming fields
(`:135-140`) in the same function; adding a type check there is the same loop. Refuse a mismatch
with `INVALID_PARAM_TYPE`, naming the parameter, the declared type and the received type — the same
shape as the existing `UNKNOWN_PARAMS` message at `:156-157`, which already lists valid parameters
and points at the wiki page. That closes it for all ~500 verbs at once and turns every existing
`RPC_PARAM_*` type string into a contract retroactively.

Two decisions belong to whoever takes it, and both should be made deliberately rather than
discovered:

1. **Coercion vs refusal.** A one-element array coerced to its single string would make the two
   `classFilter` shapes interoperate. Refusal is the safer default and coercion can be added per
   parameter; what must not happen is a third silent behaviour.
2. **Whether `asset.search` should accept an array natively.** It is a filter over a set, an array
   is the natural shape, and `blueprint.build_api_index` already spells it that way. Making
   `asset.search`'s `classFilter` accept both is a small change at `:1354` and removes the trap at
   its source. That is a separate improvement from the dispatcher fix and neither substitutes for
   the other — the dispatcher fix is what stops the *next* parameter doing this.

Independently of both: ungate `classFilterMode` on there actually being a class filter, or gate it
the same way `classFilter` is gated at `:1581`. A response should not describe how a filter was
matched when no filter was matched. That is the cheapest half of this ticket and it converts a
silent wrong answer into an obviously incomplete one.

## Same shape as

`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) carries the fullest statement of the
class: the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported. Here the deciding fact is that the filter did not run, and the
one field that would have said so — the `classFilter` echo — is suppressed by the very emptiness
that caused it.

Nearest members:

- `B-actor-list-fields-unknown-key-silently-dropped` — the closest sibling on the board: a
  parameter accepted at the boundary and dropped inside. Same silent-drop shape, different half of
  the declaration (an unknown key inside a declared param, versus a declared param of the wrong
  type).
- `E-actor-list-no-class-filter` (IN-REVIEW, Low) — the exact mirror image, and worth reading next
  to this one: there a caller guesses `classFilter` on `actor.list`, the name gate fires, and they
  get `UNKNOWN_PARAMS`. Same guess, same verb family; one is told, the other is not, and the only
  difference is whether the name happens to be declared.

## Dedup — checked, and none of these is it

`grep -ril` over the board for `classFilterMode`, `classFilter` and `asset.search`. Every hit is
adjacent, not the same defect:

- `B-asset-list-class-filter-case-divergence` (IN-REVIEW, Medium) — `asset.list`'s post-filter was
  case-sensitive where `asset.search`'s `exact` mode is case-insensitive. Its `#2` added
  `asset.search`'s `INVALID_MODE` refusal for `classFilterMode` (`AssetManageHandler.cpp:1376-
  1386`), which is worth noting for contrast: **the mode string is validated, the filter's type is
  not.**
- `B-actor-list-filter-case-mismatch` (DONE, High) — `actor.list`'s name-substring filter,
  documented case-sensitive and implemented case-insensitive.
- `B-asset-list-short-class-ensure` (DONE, Medium) — `FTopLevelAssetPath` ensure spam; results were
  always correct.
- `E-asset-list-empty-class-filter-no-diagnostic` (OPEN, Low) — closest in spirit. It investigated a
  report that `asset.list` with a class filter returned unrelated assets and **did not reproduce
  it**, concluding the likeliest cause was a key passed at the wrong nesting level. This ticket is
  the reproducible version of that suspicion on the sibling verb, with the mechanism found.
- `E-asset-search-vs-search-assets-overlap` (IN-REVIEW, Low) — verb-choice discoverability;
  enumerates `classFilter` as a param and claims nothing about its effect. (Its cited
  `AssetManageHandler.cpp:945` is ~387 lines stale.)
- `F-asset-search-native-subclass` (DONE, Medium) — the `parentClassPath` feature. Different axis.

## What was NOT done

- No source was modified.
- **The string form was not called.** That `classFilter: "StaticMesh"` filters correctly is a
  source read of `:1518-1524`, not a measurement. It is load-bearing for the title, so it is worth
  one call before fixing.
- Only `classFilter` on `asset.search` was measured. The dispatcher gap at `RpcDispatcher.cpp` is
  general by construction — no type is checked for any parameter of any verb — but no second verb
  was called to confirm the general claim behaviourally.
- No separate dispatcher ticket was filed. The general defect is stated here with its fix because
  board policy forbids umbrella tickets and one measured instance is not a survey; a fixer who
  lands the `:137` type check closes this ticket and an unknown number of unfiled ones.

severity rationale: impact=High — the README's High band verbatim, "silent wrong / stale / hardcoded data on a normal path (the caller trusts a result that is a lie and builds on it)". The call returns `success`, 38 rows, and `classFilterMode: "exact"` (`:1593`, emitted unconditionally) while the `classFilter` echo that would have exposed the drop is suppressed by the same emptiness that caused it (`:1581-1583`). The caller's next act is to pick a row and use it, and the board records exactly that happening twice already. NOT Critical: the Critical band is closed at an editor crash or a write that corrupts or loses asset data; `asset.search` is a pure read and writes nothing. NOT Medium: Medium is a soft blocker doable via a documented workaround, and there is no workaround for a result you do not know is wrong — the string spelling is a workaround only for someone who already knows this ticket exists. I weighed the argument that a caller checking the returned `class` column notices immediately, and reject it as the ordinary case: the verb's own summary sells it as "FIND AN ASSET BY NAME" (`:1332`), the query is a name match, and a caller who supplied a class filter is by definition relying on it so as not to inspect 38 rows. It is a real mitigation for a careful caller, and it is why this is not rated above High, not a reason to rate it below. x reach: bump up DECLINED, and the reason is worth recording rather than assumed. `asset.search` is a discovery verb reached in nearly every session, which is exactly the rubric's bump-up condition, and applying it mechanically lands on Critical — a band defined as crash-or-corruption that a read verb cannot satisfy. A bump that lands where the definition does not fit is a mis-rating, not a priority signal, so it is declined at the band boundary; within High this should sort early on reach. Bump down also declined — the array shape is not a rare edge path: a sibling verb declares this same parameter name as an array (`BlueprintApiIndexHandler.cpp:32`) and two existing board tickets record callers sending it. High stands.

## History
- `#1-array-typed-filter-accepted-and-ignored` `OPEN` reporter — Measured live against a running editor: `asset.search {classFilter:["StaticMesh"]}` returned 38 results, **none** a StaticMesh, with `classFilterMode:"exact"` echoed and **no** `classFilter` echoed. All line numbers re-derived this session at HEAD. **Corrects the framing this was filed under:** `classFilter` is NOT ignored wholesale — for a `string` value it is read (`AssetManageHandler.cpp:1354`) and applied (`:1518-1524`, via the handler-local `MatchesClassByMode` lambda `:1465-1482` over the registry results, loop head `:1507`); it never touches an `FARFilter` (only `ClassPaths`/`bRecursiveClasses` `:1416-1417` and `PackagePaths`/`bRecursivePaths` `:1420-1421`/`:1455-1456` are ever populated, `ClassNames` never). The defect is TYPE, not presence: declared `"string"` (`:1337`), an array flows through `Ctx.GetString` (`HandlerContext.cpp:17-24`) -> `FJsonObject::GetStringField` (`C:/UE_5.8/.../Json/Private/Dom/JsonObject.cpp:474-477`, `GetField<EJson::None>` accepts any type) -> `FJsonValue::AsString` (`.../JsonValue.cpp:26-36`), which on `TryGetString` failure logs via `ErrorMessage` (`:414`, a `LogJson` line the handler cannot see) and **returns the empty string**. `ClassFilter == ""` makes the `else if` at `:1518` false, so no filtering runs; then `:1581-1583` gates the `classFilter` echo on non-empty (suppressed) while `:1593` emits `classFilterMode` unconditionally — the response describes a match mode for a filter that never ran, which is exactly the payload measured. ROOT is dispatcher-level: `FParamSpec::Type` is declared everywhere and validated nowhere — `grep -n "Spec.Type" RpcDispatcher.cpp` gives `:52`/`:71` (`TypedAliases`, a different field) and `:107` (the missing-required-param message text); there is no `INVALID_PARAM_TYPE` path, and the gate at `:137` is `KnownParams.Contains(key)`, name-only, sending `UNKNOWN_PARAMS` at `:159`. Hence the sting: the misspelling `classNames` is refused loudly at `:159`; the correct spelling in the wrong shape is accepted silently. This is a NORMAL path, not an edge: `blueprint.build_api_index` declares the SAME name as an array (`BlueprintApiIndexHandler.cpp:32`, read via `Ctx.GetArray` `:38`), and the board records two callers sending the array shape to `asset.search` incidentally — `B-add-montage-notify-time-dropped:26` and `E-replace-node-noncallable-no-hint:68`. Fix belongs in `ValidateHandlerParams` (`:91-160`), which already walks both the specs and the fields: add the type check to that loop and emit `INVALID_PARAM_TYPE` in the shape of the existing `:156-157` message; cheapest partial fix is to gate `classFilterMode` at `:1593` the way `classFilter` is gated at `:1581`. NOT DONE: no source modified; the string form was NOT called (its correctness is a source read of `:1518-1524` and is load-bearing for the title, so verify it before fixing); only this one verb was measured, so the general dispatcher claim is source-derived, not behaviourally surveyed; no separate dispatcher ticket filed (no umbrella per board policy, and one instance is not a survey). Dedup: `grep -ril` for `classFilter`/`classFilterMode`/`asset.search` across the board — `B-asset-list-class-filter-case-divergence` (case sensitivity; its `#2` added `asset.search`'s `INVALID_MODE` check at `:1376-1386`, so the mode STRING is validated while the filter's TYPE is not), `B-actor-list-filter-case-mismatch` (name-substring case), `B-asset-list-short-class-ensure` (ensure spam, results correct), `E-asset-list-empty-class-filter-no-diagnostic` (investigated a near-identical report on `asset.list` and did NOT reproduce it), `E-asset-search-vs-search-assets-overlap` (verb overlap; its `AssetManageHandler.cpp:945` citation is ~387 lines stale), `F-asset-search-native-subclass` (`parentClassPath`). All adjacent; none claims an accepted filter has no effect. Filed under this id rather than the proposed `B-asset-search-ignores-class-filter` because the string form demonstrably works and ids are quoted verbatim by sibling tickets.
