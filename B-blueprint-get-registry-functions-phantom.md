---
id: B-blueprint-get-registry-functions-phantom
title: "blueprint.get unions registry-only functions into functions[], claiming graphs that do not exist"
status: DONE
severity: Medium
category: bug
tags: [blueprint, readback, registry, silent-wrong-data]
encounters: 1
---

# blueprint.get unions registry-only functions into functions[]

`Handlers/Blueprint/BlueprintInfoHandler.cpp:99-126` merges the in-memory blueprint registry's
`functions` array on top of the graph snapshot: any registry function whose name is absent from
`BuildBlueprintSnapshot`'s enumeration is appended wholesale. That is a positive claim that a
function graph exists on the Blueprint when it does not.

`BuildBlueprintSnapshot` sets `functions` from `CollectBlueprintFunctions`
(`BlueprintHandlerUtils.cpp:1159` area), which walks the Blueprint's function graphs, so the
snapshot is authoritative and complete. The union can therefore only add phantoms — a function
authored through `blueprint.add_function` and later removed by another verb, an editor edit, or a
`compile_bpir` sweep still reads back as present, and its registry-sourced `inputs`/`outputs` are
whatever the original request said rather than the graph's real pins.

The identical defect on the sibling `events` array was fixed under
`B-add-event-echoes-failed-pins` (the events merge now trusts the snapshot); `functions` was left
alone in that pass because the write side — which of the function-authoring verbs put what into the
registry — was not audited, and dropping the union without that audit risks removing a field some
verb depends on. The same file's `defaults` / `metadata` merges are `!Entry->HasField(...)`-gated
fallbacks and are not affected.

**Fix:** audit every writer of the registry's `functions` array, then either drop the union the way
the events merge did, or make each writer store measured graph data so the union cannot inject
anything false. Whichever way, `blueprint.get` must be an independent witness of the graph — that
is its whole job as the readback verb for the authoring family.

## History
- `#1-found-while-fixing-the-events-twin` `OPEN` reporter — Found at `b92ba268` while fixing the
  events half of the same merge block for `B-add-event-echoes-failed-pins`. Source-only finding, not
  reproduced live. Repro sketch: `blueprint.add_function`, then delete the resulting function graph
  behind the registry's back (e.g. `FBlueprintEditorUtils::RemoveGraph`, or any verb that removes
  it without clearing the registry record), then `blueprint.get` — the function is expected to still
  appear in `functions[]`.
- `#2-dropped-the-functions-union` `IN-REVIEW` developer — Premise confirmed at `7230b41d`
  (batch 4's `0c604b0f` only added `parentClass`; the union was still at
  `BlueprintInfoHandler.cpp:98-124`). Writer audit: the registry `functions` array has exactly one
  writer, `blueprint.add_function` (`BlueprintFunctionHandler.cpp` "Update registry", measured
  pins) and exactly one reader, `blueprint.get`. `blueprint.remove_function` never clears it, so
  `add_function` → `remove_function` → `get` was a public-verb repro. `BuildBlueprintSnapshot`
  always sets `functions[]` from `CollectBlueprintFunctions`, so nothing depends on the union.
  Changed the `functions` merge in `BlueprintInfoHandler.cpp` to the same `!Entry->HasField`
  fallback the events merge uses (one shared comment now covers both); the verb description,
  `docs/wiki-src/blueprint.md` (`### blueprint.get`) and `CHANGELOG.md` say `functions[]` is a
  live enumeration. The add_function registry write is now write-only state, left in place to keep
  the diff surgical. Test: `PinWright.blueprint.get.RegistryOnlyFunctionIsNotReported`
  (`Tests/Blueprint/TestAddEventPinHonesty.cpp`: add_function, `FBlueprintEditorUtils::RemoveGraph`
  behind the registry's back, assert `blueprint.get` omits it; fails with the union restored).
  Filter: `PinWright.blueprint.get.RegistryOnly`.
- `#3-verified-linux` `DONE` tester — Fix commit `3c07a5cb`. PinWright `ae877ccc` (on origin/master, base `7230b41d`), UE 5.8 Linux Vulkan. run3/full is the offscreen full suite: 5827/5827 ok, 0 fail, 73 skips, none of them this ticket's tests, and no PINWRIGHT_ASSERTIONS_SKIPPED marker for them. The build (clean unity, 0 errors) and Python (462 OK) come from run2 on the same tree. Passed non-skipped in run3/full: `PinWright.blueprint.get.RegistryOnlyFunctionIsNotReported` and its events twin `.RegistryOnlyEventIsNotReported`, plus the other seven `PinWright.blueprint.get.*` tests (`FunctionCategoryReadback`, `FunctionCategoryRoundTrips`, `ParentClassReadback`, `DefaultsReflectCdo`, `InputEventsReadback`, `OtherInputEntryKinds`, `NoComponentsContract`), with no regression. Both Fix asks are met. (1) Writer audit: the registry `functions` array has one writer (`blueprint.add_function`) and one reader (`blueprint.get`). (2) The union was dropped the way the events merge was, so `functions[]` comes only from `CollectBlueprintFunctions`. The test runs `add_function` and asserts the precondition that the registry lists the function. It then removes the graph with `FBlueprintEditorUtils::RemoveGraph` and asserts `blueprint.get` omits the function. That fails with the union restored. Coverage limit: `add_function`'s registry write is now write-only state, left in place, and nothing reads it.
