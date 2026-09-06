---
id: F-eqs-set-test-purpose-and-remove-test
title: "An EQS test's purpose (filter vs score) cannot be changed and a test cannot be removed — the only way to convert a filter into a score is to rebuild the whole query and repoint every referencer"
status: OPEN
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
