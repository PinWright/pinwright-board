---
id: E-add-variable-set-map-wrappers-undocumented
title: "blueprint.add_variable accepts set<T>/map<T,V> variableType wrappers but advertises only array<T> — agents must read parser source to discover them"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, discoverability, blueprint, add_variable, container]
---

# `blueprint.add_variable` accepts `set<T>` / `map<T,V>` but neither the param doc nor the `TYPE_NOT_FOUND` error lists them

`blueprint.add_variable`'s `variableType` token grammar (`BpirTypeSpecParser.cpp`
lines 54-85) recognizes three container wrappers — `array`, `set`, and `map` —
mapping them to `EPinContainerType::Array` / `Set` / `Map`. So declaring a member
variable as a `TSet<FName>` via `variableType:"set<name>"` (and a `TMap` via
`map<K,V>`) **works**. But the tool's self-description advertises only the array
form:

- The param description (`BlueprintPropertyHandler.cpp:37`) lists primitives and
  the `class:`/`struct:` ref forms, and names **no** container wrapper at all.
- The `TYPE_NOT_FOUND` error (`BlueprintPropertyHandler.cpp:85-90`) enumerates the
  accepted wrappers as
  `array<T>, object<T>, struct<T>, enum<T>, class<T>, softobject<T>, softclass<T>, interface<T>`
  — `array<T>` is the **only** container kind shown; `set<T>` and `map<T,V>` are
  omitted despite being accepted.

Because `set`/`map` are absent from both the advertised grammar and the failure
message, a caller asked to add a `TSet`/`TMap` member has no in-product signal
that the wrapper exists. The token resolves fine once guessed, but it is not
discoverable: the only authoritative confirmation is the parser source.

## Evidence (this task, `container.set.contains` namespace)

Friction note, verbatim: *"Discoverability gap: blueprint.add_variable's doc and
its TYPE_NOT_FOUND error only advertise array<T> among container wrappers, not
set<T>/map<T,V>; I had to read plugin C++ (BpirTypeSpecParser.cpp) to confirm
set<name> is accepted — it is, and worked."*

The task created `/Game/Fuzz/BP_RegionTracker` with a public `TSet<FName>`
`UnlockedRegions` (`add_variable` `set<name>` → succeeded, compiled clean), so the
capability is real and the only cost was the out-of-band source read to confirm
the token before issuing the call. Pure discoverability overhead, no functional
failure on this path.

## What it should do

- The `variableType` param description should name the container wrappers it
  accepts, including `set<T>` and `map<K,V>` (not just imply `array<T>`).
- The `TYPE_NOT_FOUND` error's wrapper list should include `set<T>` and
  `map<K,V>` alongside `array<T>`, so a failed attempt self-documents the full
  accepted grammar.
- The `blueprint.md` wiki overlay's `add_variable` section should show a
  `set<T>` / `map<K,V>` example so callers don't have to discover it from the
  parser. (`blueprint.function`'s `inputs[].type` reuses the same token grammar —
  `BlueprintFunctionHandler.cpp:69` — so the doc fix benefits that path too.)

This is docs/strings only; the parser already supports the tokens. NOT the
`/`-leading path / `class:`/`struct:` regression of `E-add-variable-type-format`
(that is about advertised forms being *rejected*); here the forms *work* but are
*unadvertised*. NOT `E-container-set-no-discoverable-target` (that is about
finding a `TSet` *target object/property* for the `container.set` verbs, not
about declaring a `set<T>` *variable type*). NOT the corruption bug
`B-container-set-add-remove-no-rehash` (the functional defect filed by the judge
for this same task).

**Fix:** Downstream — edit the `variableType` param string and the
`TYPE_NOT_FOUND` wrapper list in `BlueprintPropertyHandler.cpp` to include
`set<T>`/`map<K,V>`, and add a `set`/`map` wrapper example to
`docs/wiki-src/blueprint.md`'s `add_variable` section. Docs/strings only; no
parser change.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of a `container.set.contains`
  lifecycle task. `add_variable` `set<name>` succeeded and the BP compiled clean,
  but the reporter had to read `BpirTypeSpecParser.cpp` to confirm `set<...>` was
  accepted because neither the `variableType` param doc
  (`BlueprintPropertyHandler.cpp:37`, names no container wrapper) nor the
  `TYPE_NOT_FOUND` error (`:85-90`, lists only `array<T>` among containers)
  advertises `set<T>`/`map<T,V>` — even though the parser maps all three
  (`array`/`set`/`map`, `BpirTypeSpecParser.cpp:54-85`). Distinct
  discoverability/docs angle from `E-add-variable-type-format` (rejected forms),
  `E-container-set-no-discoverable-target` (finding a target object), and the
  judge-filed corruption bug `B-container-set-add-remove-no-rehash`.
- `#2-corroborated-map-variant` `OPEN` reporter — Same discoverability gap hit
  again from a `container.map` lifecycle task (BP_ResourceCosts, `TMap<FString,int32>`).
  This task used the **`map<string,int>`** wrapper specifically — the exact token
  this ticket's title/body call out as accepted-but-unadvertised. Friction note,
  verbatim: *"neither the blueprint.add_variable wiki page nor its TYPE_NOT_FOUND
  error message lists the map<K,V> wrapper (they list array/object/struct/enum/
  class/softobject/softclass/interface only), so I had to read the plugin C++
  type-spec parser (BpirTypeSpecParser/IrTypeSpecParser) to confirm variableType
  'map<string,int>' is accepted — it worked, but only after source diving."*
  Cross-task corroboration: the `set<T>` variant produced #1 from a `container.set`
  task; the `map<K,V>` variant reproduces the identical source-dive overhead here.
  Both wrappers parse fine; both remain absent from the param doc and the
  `TYPE_NOT_FOUND` list. (The functional corruption on this task is the separate
  judge-filed `B-container-map-remove-no-rehash`; this entry is the docs/strings
  angle only.)
- `#3-document-set-map-wrappers` `IN-REVIEW` developer — Docs/strings only, no
  parser change (the parser already maps `array`/`set`/`map` →
  `EPinContainerType` in `BpirTypeSpecParser.cpp`). Three edits in the plugin
  clone: (1) `BlueprintPropertyHandler.cpp` `variableType` param description now
  names the container wrappers `array<T>, set<T>, map<K,V>` with `set<name>` /
  `map<string,int>` examples; (2) the `TYPE_NOT_FOUND` wrapper list in the same
  handler now leads with `array<T>, set<T>, map<K,V>, ...` so a failed attempt
  self-documents the full accepted grammar; (3) `Docs/wiki-src/blueprint.md`'s
  `### blueprint.add_variable` section gains a `variableType` containers
  paragraph + a `set<name>` example JSON (and a pointer to `call("bpir.types")`),
  which also covers `blueprint.add_function`'s `inputs[].type` since it reuses the
  same grammar. Regression test added in
  `Source/EditorAutomationRpcGateway/Private/Tests/Core/TestMakePinType.cpp`
  (`FMakePinTypeSetMapContainerWrappersTest`,
  `EditorAutomationRpcGateway.core.make_pin_type.SetMapContainerWrappers`): pins
  both halves of the contract — the production parse path resolves `set<name>` to
  a `Set`-container `PC_Name` pin and `map<string,int>` to a `Map`-container pin
  with `PC_String` key / `PC_Int` value, AND the registered `blueprint.add_variable`
  `variableType` param description (read from `FAutoRegisterHandler::GetPendingRegistrations()`)
  contains the `set<` and `map<` substrings, so reverting the param-string edit
  fails the test.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
