---
id: B-focus-actor-empty-string-uncovered
title: "editor.focus_actor's empty-string guard is unreachable by any test — the dispatcher gate checks presence, not emptiness, and the registry walk supplies a non-empty sentinel"
status: OPEN
severity: Low
category: bug
tags: [editor, focus_actor, required-param, dispatcher-gate, empty-string, test-coverage, registry-walk]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# A correct guard that nothing exercises

`editor.focus_actor` declares `actorName` as `RPC_PARAM_REQ`
(`Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp:141`), but the dispatcher's
required-param gate tests **presence only**: `PayloadHasParamOrAlias`
(`Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:33-58`) reduces to
`Params->HasField(Spec.Name)` at `:40` plus two alias loops (`:45-58`), so `{"actorName": ""}`
satisfies `ValidateHandlerParams` (`:91-164`) and reaches the handler body.

The body refuses it separately — `ViewportHandler.cpp:144-148`:

```cpp
FString ActorName = Ctx.GetString(TEXT("actorName"));
if (ActorName.IsEmpty()) { Ctx.SendError(TEXT("INVALID_ARGUMENT"), TEXT("actorName required")); return true; }
```

No test reaches that branch. The consolidated registry walk
`PinWright.infra.contract.RequiredParamGate.EveryVerb`
(`Source/PinWright/Private/Tests/Infra/TestContractConsistency.cpp:449-607`) leaves the probed slot
**absent** rather than empty and fills prior slots with `SetPlaceholder` (`:330-353`), whose string
branch writes the non-empty sentinel `__pinwright_required_param_probe__` (`:351`). `actorName` is
the verb's only required slot at index 0, so the walk dispatches a fully empty payload and stops at
the dispatcher gate. The two `editor.focus_actor` tests both send non-empty values:
`ValidParamsNoCrash` sends `"DirectionalLight_0"`
(`Source/PinWright/Private/Tests/EditorOps/TestEditorHandlers.cpp:1087`) and
`ResolvesInternalName` sends a real spawned actor's internal name (`:1158`). Repo-wide
`grep 'SetStringField(TEXT("actorName"), TEXT(""))'` over `Source/` returns **zero** hits.

The behaviour itself is correct — the caller gets a clean `INVALID_ARGUMENT`. This is missing
coverage of a documented guard, not a live product defect.

## Why it is filed here rather than left in the backlog

`Docs/plans/defect-backlog.md:932-935` already records it verbatim as a "**Follow-up (new, not part
of this defect)**" note under `D-8E`: "`editor.focus_actor` accepts `actorName: ""` through the
dispatcher gate and is refused only by the in-body check at `ViewportHandler.cpp:145-147`. Nothing
covers that path: the registry walk supplies placeholders rather than empty strings. An
`EmptyStringParam` test would be honest coverage; it needs a build window, so it is not in this
pass." Foreshadowed again at `:921-923`.

`D-8E`'s own Status is **FIXED**, so the note sits inside a closed entry with no independent status
and no owner and will not survive that entry being cleared. That is the reason for a board ticket
rather than a second backlog row.

## The general shape is worth a decision

The presence-vs-emptiness gap is not specific to this verb: `RPC_PARAM_REQ` guarantees the key
exists, never that it carries a usable value, so every verb enforcing non-emptiness in-body has the
same untested branch. The backlog note at `:921-923` already observes that a `MissingRequiredParam`-
named test structurally cannot pin it. Worth deciding once whether the gate should reject empty
strings for `RPC_PARAM_REQ` slots, rather than adding one test per verb.

**Fix:** a `PinWright.editor.focus_actor.EmptyStringParam` test asserting `INVALID_ARGUMENT` for
`{"actorName": ""}`. Note the id must not dot-extend an existing id — see
`B-test-ids-swallowed-by-dot-prefix`. Separately, `ValidParamsNoCrash`
(`TestEditorHandlers.cpp:1080-1090`) asserts only `TestTrue("handler found", ...)` and is an instance
of the `D-8C` vacuous pattern; the new test is the honest replacement for it.

## Related

- `Docs/plans/defect-backlog.md` `D-8E` (FIXED) — carries this as an unowned follow-up note.
- `Docs/plans/defect-backlog.md` `D-8C` (CONFIRMED) — vacuous `ValidParamsNoCrash` tests.
- `E-focus-actor-rejects-internal-name-label-only` (IN-REVIEW) — the internal-name resolution defect,
  already fixed and regression-tested at `TestEditorHandlers.cpp:1107`. Different defect.

## History
- `#1-guard-correct-branch-untested` `OPEN` reporter — `editor.focus_actor` declares `actorName` as `RPC_PARAM_REQ` (`ViewportHandler.cpp:141`), but the dispatcher gate tests presence only — `PayloadHasParamOrAlias` (`RpcDispatcher.cpp:33-58`) reduces to `Params->HasField(Spec.Name)` at `:40` — so `{"actorName": ""}` satisfies `ValidateHandlerParams` (`:91-164`) and reaches the body, which refuses it separately at `ViewportHandler.cpp:144-148` with `INVALID_ARGUMENT`. No test reaches that branch: the registry walk `RequiredParamGate.EveryVerb` (`TestContractConsistency.cpp:449-607`) leaves the probed slot absent rather than empty and fills prior slots with the non-empty sentinel `__pinwright_required_param_probe__` (`SetPlaceholder`, `:330-353`, string branch `:351`), and `actorName` is the verb's only required slot, so the walk dispatches an empty payload and stops at the gate; the two `editor.focus_actor` tests send `"DirectionalLight_0"` (`TestEditorHandlers.cpp:1087`) and a real internal name (`:1158`), and `grep 'SetStringField(TEXT("actorName"), TEXT(""))'` over `Source/` returns zero hits. The behaviour is correct — this is missing coverage of a documented guard, not a product defect. Already written down at `Docs/plans/defect-backlog.md:932-935` as a follow-up note under `D-8E` (and foreshadowed at `:921-923`), but `D-8E`'s Status is FIXED, so the note carries no independent status or owner and will not survive that entry being cleared — hence a board ticket rather than a second backlog row. The presence-vs-emptiness gap is general: `RPC_PARAM_REQ` guarantees the key exists, never a usable value, so every verb enforcing non-emptiness in-body has the same untested branch, and the backlog note at `:921-923` observes that a `MissingRequiredParam`-named test structurally cannot pin it — worth one decision rather than one test per verb. Fix: add `PinWright.editor.focus_actor.EmptyStringParam` asserting `INVALID_ARGUMENT`, choosing an id that does not dot-extend an existing one (see `B-test-ids-swallowed-by-dot-prefix`); it is also the honest replacement for `ValidParamsNoCrash` (`TestEditorHandlers.cpp:1080-1090`), which asserts only that the handler was found and is an instance of the `D-8C` vacuous pattern.
