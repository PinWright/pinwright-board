---
id: F-eqs-set-test-purpose-and-remove-test
title: "An EQS test's purpose (filter vs score) cannot be changed and a test cannot be removed — the only way to convert a filter into a score is to rebuild the whole query and repoint every referencer"
status: DONE
severity: Medium
category: feature
tags: [eqs, test, purpose, filter, score, mutation]
---

# No `eqs.set_test_purpose`, no `eqs.remove_test`

`eqs.add_test` takes `purpose` (`filter` | `score` | `filter_and_score`) at creation time. After
that the purpose is fixed: no verb changes it, and no verb removes a test. Tuning an existing query
therefore cannot express the most ordinary EQS edit there is — "this filter is rejecting everything,
make it a score instead".

Both fallbacks are refused from Python.

    t = unreal.find_object(option, 'EnvQueryTest_Trace_0')
    t.get_editor_property('test_purpose')
    -> Exception: EnvQueryNode: Failed to find property 'test_purpose' for attribute
       'test_purpose' on 'EnvQueryTest_Trace'

    t.set_editor_property('context', new_context_class)
    -> Exception: EnvQueryNode: Property 'Context' for attribute 'context' on
       'EnvQueryTest_Trace' cannot be edited on instances

(The second is why `eqs.set_context_class` has to exist at all; the first has no verb equivalent.)

`eqs.set_test_filter` is not a way out: its schema is a closed branch — `match`/`bool` with a boolean
value, or `minimum`/`maximum`/`range` with numbers. For a bool test like `Trace` every setting still
rejects one side; there is no "stop filtering" value.

## What it costs

A cover query in this project used two `Trace` tests as **filters** ("blocked at crouch height" AND
"clear at standing height", both toward a context). When the context legitimately resolves to the
querier's own location — which happens whenever the AI has no target, i.e. exactly when it needs to
find cover to reload behind — those two conditions cannot both hold, so the query returned **zero
results in 348 consecutive samples** and every branch depending on cover silently failed.

The one-line fix is "make those two tests scoring". Instead it requires:

1. `eqs.create` a second query asset (there is no `overwrite` on `eqs.create`),
2. re-add the generator and every test with the right `purpose`,
3. re-apply every context, filter and scoring setting,
4. find and repoint every referencer — here three `BTTask_RunEQSQuery` nodes in a Behavior Tree,
5. and either leave the original as dead content or delete it, where
   `asset.delete force:true` is banned in this project for unrelated reasons.

That is a lot of moving parts and several chances to leave the tree pointing at the old query, for
an edit that is a single enum in the EQS editor.

## Ask

1. **`eqs.set_test_purpose {queryPath, generatorIndex, testIndex, purpose}`** — same
   `filter | score | filter_and_score` vocabulary `add_test` already accepts.
2. **`eqs.remove_test {queryPath, generatorIndex, testIndex}`** — so a query can be corrected rather
   than rebuilt. Test indices shifting on removal is fine and expected; say so in the page.
3. Lower priority, but it removes the other half of the rebuild: an `overwrite` flag on `eqs.create`,
   matching `blueprint.create`'s, so a query can be regenerated in place without breaking referencers.

## Notes

- Follow-on to `F-eqs-namespace-expansion` (DONE), which delivered `eqs.create`, `add_generator`,
  `add_test`, `set_context_class`, `set_test_filter` and `set_test_scoring`. That set covers
  authoring a query forwards; this is the missing "edit one that already exists" half. Its own
  use-case list includes "Tuning scoring/filter behavior without opening the EQS editor", which is
  the thing that is still not possible when the tuning crosses the filter/score line.
- Adjacent: `E-eqs-readback-asset-dump-elides-tests` — reading a query's tests back is already
  awkward, which compounds this: a caller cannot easily confirm which index is which purpose before
  mutating.

## History
- `#1-purpose-remove-overwrite` `IN-REVIEW` developer — Added `eqs.set_test_purpose {queryPath, generatorIndex=0, testIndex, purpose, save}` (strict `filter`/`score`/`filter_and_score`, unknown value is `INVALID_ARGUMENT` with the test untouched; response `previousPurpose`, `purpose` read back, `changed`, measured save report) and `eqs.remove_test {queryPath, generatorIndex=0, testIndex, save}` (later indices shift down and their `TestOrder` is renumbered; when the query has an EQS editor graph the matching test subnode is removed too, because `UEnvironmentQueryGraph::UpdateAsset` rebuilds `Tests` from the graph and would otherwise restore the test; response `removedTestIndex`, `removedTestClass`, `testCount`, `editorGraphNodesRemoved`). Ask 3: `eqs.create` takes `overwrite` (default false) that clears an existing query's options in place (same object, referencers stay valid), closes any open EQS editor for it and drops its editor graph; the `ai.create_eqs_query` shim does not gain it. `eqs.add_test`'s lenient purpose parse (unknown → score) is unchanged. Files: `Source/PinWright/Private/Handlers/AI/EQSHandler.cpp`, `.../AI/EQSHandler.h`, `Source/PinWright/Private/Tests/Gameplay/TestEQSHandlers.cpp`, `docs/wiki-src/eqs.md`, `CHANGELOG.md`. Tests: `PinWright.eqs.SetTestPurposeConvertsFilterToScore`, `PinWright.eqs.RemoveTestSurvivesEditorGraphRebuild` (builds the real EQS editor graph the way the editor's first open does, removes index 1, then runs the graph's `UpdateAsset` and asserts the test stays gone), `PinWright.eqs.CreateOverwriteClearsInPlace`; fixtures under `/Game/PinWrightTests/EQS`. Compile-checked only (clang syntax check); needs a build and run.
- `#2-review-fixes` `IN-REVIEW` developer — Review findings applied. (1) `eqs.create overwrite:true` now reports `closedEditorCount` (from `FindEditorsForAsset` before the close) and `editorGraphDropped`, and its save is measured through `SendMutationWithSaveReport` (`saveRequested`/`saved`/`saveState`/`saveDetail`, `SAVE_FAILED` when the write does not land) instead of echoing `save` after a mark-dirty; a fresh create is unchanged (filed `B-eqs-authoring-verbs-saved-echoes-request` for it and the sibling verbs). (2) `PinWright.eqs.CreateOverwriteClearsInPlace` now builds the real EQS editor graph before the overwrite and asserts `Query->EdGraph == nullptr`, `editorGraphDropped:true`, `closedEditorCount:0`, `saveState:"written"` and a clean package afterwards. NITs: `eqs.set_test_purpose` skips the transaction and dirtying when the purpose is unchanged; comments on the `TestOrder` vs `UpdateAsset` subnode-index gap and on why the editor close is not tick-unsafe. The graph build is shared as `TestEqsTestEditingHelpers::BuildEditorGraph`. Files: `EQSHandler.cpp`, `TestEQSHandlers.cpp`, `docs/wiki-src/eqs.md`, `CHANGELOG.md`. Compile-checked (clang syntax) only; needs a build and run.
- `#3-verified-linux` `DONE` tester — Fix commit 2742d856. Passed non-skipped in run3/full: `PinWright.eqs.SetTestPurposeConvertsFilterToScore` (ask 1), `PinWright.eqs.RemoveTestSurvivesEditorGraphRebuild` (ask 2: the test stays gone after the EQS editor graph's UpdateAsset rebuild) and `PinWright.eqs.CreateOverwriteClearsInPlace` (ask 3: same object cleared in place, editor graph dropped, measured save). Existing `PinWright.eqs.*` authoring tests also passed. Docs verified at 7230b41d: `docs/wiki-src/eqs.md` `### eqs.remove_test` states that indices shift, as the ask requested, and the `eqs.create` overwrite paragraph explains in-place clearing and the difference from blueprint.create. All three asks are met. Limit: the reporter's cover query and Behavior Tree were not replayed; the fixtures are under /Game/PinWrightTests/EQS. The fresh-create `saved` echo is tracked separately in B-eqs-authoring-verbs-saved-echoes-request.
