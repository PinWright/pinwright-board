---
id: E-metasound-variable-authoring-docs-gap
title: "MetaSound variable-authoring RPCs + their variableType registry spellings are absent from the audio.authoring wiki overlay"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, metasound, audio, authoring, add_metasound_variable, variableType, discoverability]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `add_metasound_variable` & siblings are undocumented on the `audio.authoring` wiki overlay

The `audio.authoring.md` wiki overlay walks through `create_metasound`,
`add_metasound_input`, `add_metasound_node`, `add_metasound_output`,
`connect_metasound_nodes`, `set_metasound_default`, and `describe_metasound`,
but has **no section** for the graph-*variable* authoring RPCs the task above
exercised: `add_metasound_variable`, `set_metasound_variable_default`, and
`remove_metasound_variable`. Nothing on the overlay tells a caller these RPCs
exist, what `variableType` values they accept, or that the accepted type
spellings differ from the natural names. The companion
`audio.authoring.metasound_gotchas.md` page (the page that exists precisely to
warn about MetaSound engine-API traps) does not mention the variable-type
registry-spelling trap either.

## What's awkward

A technical-sound-designer task ("add a Float/Int/Bool/String graph variable")
copies `variableType` from the only place it is documented — the per-method
wiki page generated from the handler param spec, which advertises
`Type name (Float, Int, Bool, String)`. Three of those four work; `Int` is
rejected with a generic `[VARIABLE_FAILED]` that names no accepted alternative.
There is no overlay prose, no gotchas entry, and no error hint pointing at the
working spelling (`Int32`), so the only path to recovery is reading the plugin
handler and `MetaSoundLiteralFromTypeName.cpp` to discover the registry keys
integers as `Int32`. That diagnose-by-source-reading detour is the friction.

## What it should do

This is the **process/discoverability complement** to the code bug
`B-metasound-variable-int-type-rejected` (which the per-finding judge filed to
fix the handler so the documented `Int` actually maps to the registry's
`Int32`). Independent of how that code bug resolves, the docs should let a
caller learn the variable-authoring surface without source-reading:

- Add an `add_metasound_variable` / `set_metasound_variable_default` /
  `remove_metasound_variable` section to `docs/wiki-src/audio.authoring.md`,
  alongside the existing input/node/output workflow, naming the `variableType`
  contract and the typical add → set-default → remove → describe sequence the
  task used.
- Add a one-line trap to `docs/wiki-src/audio.authoring.metasound_gotchas.md`:
  graph-variable types resolve against the MetaSound data-type registry, whose
  canonical key for a 32-bit integer is `Int32` (and any other documented alias
  the registry spells differently, e.g. `Bool` vs `Boolean`) — keep this note
  in sync with `B-metasound-variable-int-type-rejected`'s resolution (drop the
  `Int32` caveat once the handler maps the documented `Int` alias).

The edit itself is a downstream wiki process, not a code change. Named overlay
pages: `docs/wiki-src/audio.authoring.md` and
`docs/wiki-src/audio.authoring.metasound_gotchas.md`.

## Evidence (this task — `MS_AmbientWind` under `/Game/Audio/MetaSounds`)

5 up-front `wiki-nav` reads, then the add sequence ran into the type trap. From
the friction note: *"the wiki's documented variableType 'Int' was REJECTED with
[VARIABLE_FAILED] — add_metasound_variable passes the raw type string to
Builder.AddGraphVariable, whose registry keys integers as 'Int32', not 'Int'
(the wiki and the literal-helper's accepted aliases both say 'Int', a real
contract/discoverability bug). I diagnosed it by reading the plugin handler +
MetaSoundLiteralFromTypeName.cpp and recovered by retrying with
variableType='Int32'."* Call log: `add_metasound_variable WindLayerCount Int 3`
→ `is_error: [VARIABLE_FAILED]` (1 failed call), then
`add_metasound_variable WindLayerCount Int32 3` → ok. Trial-and-error retry on
one method whose only documentation (the auto-generated param spec) advertises
the spelling that fails, with no overlay/gotchas page to consult first. Mirrors
the accepted `E-add-variable-category-param-undocumented` precedent (docs/
discoverability complement filed alongside a code bug for the same RPC family).

## History
- `#1-initial-audit` `OPEN` reporter — Process/discoverability finding from the `MS_AmbientWind` MetaSound-variable authoring task. The `audio.authoring.md` overlay documents create/input/node/output/connect/describe but has no section for `add_metasound_variable` / `set_metasound_variable_default` / `remove_metasound_variable`, and neither overlay nor `metasound_gotchas.md` mentions that `variableType` resolves against the data-type registry (integer key = `Int32`, not the documented `Int`). Caller hit `[VARIABLE_FAILED]` on the documented `Int` form (1 failed call), then recovered only after reading the plugin handler + `MetaSoundLiteralFromTypeName.cpp` and retrying with `Int32`. Distinct PROCESS angle from the code bug `B-metasound-variable-int-type-rejected` (per-finding judge's filed_id, which proposes mapping the alias in the handler): this asks for the variable-authoring RPCs + the registry-spelling trap to be documented on the `audio.authoring` wiki overlay / gotchas page regardless of how the code bug resolves. Deduped against `E-add-variable-type-format` (DONE — `blueprint_add_variable`, different namespace) and `E-add-variable-category-param-undocumented` (different param/RPC family); genuinely new.
</content>
</invoke>
