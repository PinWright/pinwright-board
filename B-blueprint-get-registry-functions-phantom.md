---
id: B-blueprint-get-registry-functions-phantom
title: "blueprint.get unions registry-only functions into functions[], claiming graphs that do not exist"
status: OPEN
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
