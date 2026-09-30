---
id: B-remove-event-inputkey-ignores-modifiers
title: "blueprint.remove_event matches InputKey events by key only: `key_pressed J` removes both J and Ctrl J, and the BPIR spelling `key_pressed J(ctrl)` matches nothing"
status: IN-REVIEW
severity: Medium
category: bug
tags: [remove-event, inputkey, k2node-inputkey, modifiers, bpir]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
---

# remove_event cannot tell J from Ctrl J

BPIR now spells InputKey modifiers inside the entry parens (`entry key_pressed J(ctrl)`, see
`B-bpir-inputkey-modifiers-dropped`). `blueprint.remove_event` did not follow:
`ParseInputKeyRemoveEventName` in `Handlers/Blueprint/BlueprintEventHandler.cpp` strips a
`key_pressed ` / `key_released ` prefix and a trailing `()` only, and the node match uses
`FBpirInputKeyHelpers::DoesInputKeyMatchBpirIdentifier`, which compares the key alone.

## Observed (code reading, not yet run live)

- `eventName: "key_pressed J"` on a Blueprint holding both a plain J and a Ctrl J InputKey node
  selects both roots, so both subgraphs are removed unless the caller passes `nodeId`.
- `eventName: "key_pressed J(ctrl)"` (the exact text `blueprint.decompile` prints) leaves the
  identifier as `J(ctrl)`, which matches no node, so the call reports nothing to remove.

**Workaround:** pass `nodeId` (the InputKey node GUID from `blueprint.graph.find_nodes`).
**Fix (proposed):** parse an optional `(mods)` suffix with the same token set as the BPIR parser and
match with `FBpirInputKeyHelpers::FormatInputKeyModifiers`, the way `IsInputKeyNodeForBpirEntry` does.

## History
- `#1-found-during-modifier-fix` `OPEN` developer - Found while fixing `B-bpir-inputkey-modifiers-dropped`: compile_bpir identity now includes modifiers, remove_event's does not. Left out of that fix to keep it to the BPIR grammar.
- `#2-remove-event-matches-modifiers` `IN-REVIEW` developer - `ParseInputKeyRemoveEventName` (`Handlers/Blueprint/BlueprintEventHandler.cpp`) now splits a trailing `(mods)` list with the shared `FBpirInputKeyHelpers::ParseInputKeyModifiers` (the BPIR parser uses the same function now), and the node match goes through the new `FBpirInputKeyHelpers::DoesInputKeyMatchBpirKey` (key plus modifiers), which `IsInputKeyNodeForBpirEntry` also uses. `J`, `J()` and `key_pressed J` match only the unmodified key; `key_pressed J(ctrl)` matches only Ctrl J; an unparseable list leaves the identifier whole, so it matches no key. `eventName` param description and the `blueprint.remove_event` section of `docs/wiki-src/blueprint.md` updated. Test: `PinWright.blueprint.remove_event.InputKeyMatchesModifiers` (plain J + Ctrl J; `key_pressed J` must leave Ctrl J, then `key_pressed J(ctrl)` must remove it).
