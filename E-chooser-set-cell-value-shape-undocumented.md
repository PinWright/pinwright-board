---
id: E-chooser-set-cell-value-shape-undocumented
title: "chooser.set_cell value shape per column kind ({min,max} float, {value,comparison} object/enum) is undocumented — forces an engine/handler source dive"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [chooser, docs, set_cell, discoverability]
---

# `chooser.set_cell` value shape per column kind is undocumented

`chooser.set_cell` takes a single `value` param declared in the RPC spec
only as `RPC_PARAM_REQ("value", "any", "Typed cell value.")`. The actual
JSON shape that `value` must take depends on the stored column kind, and
each shape is different and non-obvious:

- **float (range) column** — an object `{min, max}` (either bound may be
  omitted for an open-ended range), or a bare number for a point match.
- **bool column** — a bool literal, or a `MatchAny` sentinel.
- **object column** — an object `{value, comparison}` (asset/object path
  + a comparison token like `MatchAny`/equal).
- **enum column** — an object `{value, comparison}` (enum token + a
  comparison token).

None of these shapes appear in the `docs/wiki-src/chooser.md` overlay
(it is a single descriptive sentence) nor in the generated RPC method
reference (the param is just `"any"`, "Typed cell value."). An agent that
correctly sets up the table (context class, bound columns, rows) still
hits a wall at cell population: there is no documented way to learn that a
float cell wants `{min,max}` while an object cell wants
`{value,comparison}`. The only way to discover the contract is to read
the handler C++
(`Source/PinWrightChooser/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp`,
`SetFloatCell` / bool / `SetObjectCell` / enum branches).

This is **distinct** from
[E-chooser-column-binding-undocumented](E-chooser-column-binding-undocumented.md):
that ticket documents the column → context → `propertyBinding` *authoring
contract* (which columns need bindings, and the build order). It says
nothing about the per-kind JSON *value shape* that `set_cell` expects.
The two together cover "how do I bind a column" (existing ticket) and
"what JSON do I pass to fill a bound cell" (this ticket).

## What it should do

Document the per-column-kind `set_cell` value shapes on the
`docs/wiki-src/chooser.md` overlay (downstream wiki process), ideally
under a `### chooser.set_cell` H3 so it renders on direct `call()`:

- A small table mapping column kind → accepted `value` shape:
  - float → `{min, max}` (bound optional) or a bare number
  - bool → `true`/`false` or `MatchAny`
  - object → `{value: "<asset/object path>", comparison: "MatchAny"|"equal"|...}`
  - enum → `{value: "<enum token>", comparison: "MatchAny"|"equal"|...}`
- Note that the column kind is authoritative from `add_column`, so the
  same `value` param is dispatched differently per column.
- Optionally tighten the RPC param description from the bare
  `"any" / "Typed cell value."` to name the per-kind shapes (or point to
  the overlay).

## Evidence

From this task's friction note (focus `chooser`, ~44 MCP calls, built
`CT_HitReaction` end to end): the agent set 9 cells across 3 columns
(float `{50,100}`/`{0,50}`/`{0,100}`, bool true/true/false, object
any/null) successfully — but only after determining the shapes by
reading source. Quoting the note: *"The set_cell float-range shape
({min,max}) and object cell shape ({value,comparison}) are not in the
wiki and I had to read the plugin handler C++ to confirm them (noted
discoverability gap)."* The final `chooser.compile` returned
`diagnostics:[] hasDiagnostics:false`, so the authoring was correct — the
friction was purely the undocumented value-shape discovery, an avoidable
source dive on top of the column-binding source dive already tracked in
E-chooser-column-binding-undocumented.

This is a **docs/discoverability** gap (the tool works once the shape is
known), not a tool bug, and is separate from the judge-filed
`E-add-variable-type-format` (which is about the
`blueprint.add_variable` `variableType` token for object references).

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of task focus
  `chooser` (~44 calls): the agent successfully built `CT_HitReaction`
  but had to read
  `Source/EditorAutomationRpcGateway/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp`
  to learn that `chooser.set_cell`'s `value` (declared only as `"any" /
  "Typed cell value."`) wants `{min,max}` for a float column and
  `{value,comparison}` for an object/enum column — neither shape is in
  the `docs/wiki-src/chooser.md` overlay nor the generated RPC reference.
  Distinct from E-chooser-column-binding-undocumented (column→context→
  propertyBinding contract) and from the judge-filed
  E-add-variable-type-format (the `blueprint.add_variable` object-ref
  token). Proposes a per-column-kind value-shape table on the chooser
  overlay (and optionally tightening the `set_cell` param description).
- `#2-fix` `IN-REVIEW` developer — Documented the per-column-kind
  `chooser.set_cell` `value` shapes. Added a `### chooser.set_cell` H3
  section to `Docs/wiki-src/chooser.md` with a kind→shape table: float =
  `{min, max}` (either bound omittable) or a bare number point-match;
  bool = JSON bool or `"true"`/`"false"`/`"any"` token (default
  `MatchAny`); object = `{value/assetPath, comparison}` or a bare path
  string; enum = `{value, comparison}` (resolved against the column's
  `enumType`); randomize = a bare weight number. All shapes verified
  against `ChooserAuthoringHandler.cpp` (`SetFloatCell` :501-546,
  `SetBoolCell` :467-499, `SetObjectCell` :684-714, `SetEnumCell`
  :611-682, dispatched by `SetCellValue` :723-762). Also tightened the
  RPC param description at `ChooserAuthoringHandler.cpp:1223` from the
  bare `"any" / "Typed cell value."` to name the per-kind shapes and
  point at the wiki page. Regression test
  `FWikiHandlerSetCellDocumentsValueShapesTest`
  (`EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.SetCellDocumentsValueShapes`,
  in `Private/Tests/Infra/TestWikiHandler.cpp`) renders
  `chooser.set_cell` via `WikiHandler::RenderPage` and asserts the
  overlay-exclusive markers `{min, max}` (with a space — the param desc
  uses the spaceless form) and the object `assetPath` alias appear; both
  vanish if the H3 section is reverted. Files: `Docs/wiki-src/chooser.md`,
  `Source/.../Handlers/Chooser/ChooserAuthoringHandler.cpp`,
  `Source/.../Tests/Infra/TestWikiHandler.cpp`.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp` → `Source/PinWrightChooser/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
