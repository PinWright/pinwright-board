---
id: E-chooser-column-binding-undocumented
title: "chooser wiki never says input columns need a context object + propertyBinding to compile clean"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [chooser, docs, add_column, compile, property-binding, discoverability]
---

# `chooser.*` wiki omits the column→context→propertyBinding contract, forcing a full delete-and-rebuild

`chooser.add_column(kind=float|bool)` accepts a column with no
`propertyBinding` and `chooser.create` accepts no `contextObjectType`,
and both calls succeed silently. But a float/bool **input** column has
nothing to evaluate against unless the table has a context object class
(`contextObjectType` on `create`) **and** each column carries a
`propertyBinding` resolving a property on that context class. Without
both, `chooser.compile` emits one `No Property Bound` error diagnostic
per unbound column — but only at compile time, after the whole table is
already built. Nothing on `create` or `add_column` warns that the column
is non-functional, and the wiki page does not mention the contract at
all, so the failure is discovered late and the recovery is expensive.

The `docs/wiki-src/chooser.md` overlay page is a single-sentence stub: it
describes the verbs but never states (a) that input columns require a
context object + per-column `propertyBinding` to compile without errors,
(b) that `contextObjectType` is effectively mandatory for any
property-driven column rather than an optional convenience, or (c) that
"viewer distance"/"is hero"-style semantic inputs have no stock-class
property and must be mapped onto whatever the chosen context class
exposes. An agent reading only the wiki will build the table the obvious
way (create with no context, add float/bool columns, set cells) and only
learn it's wrong from the compile diagnostics.

## What it should do

Document the column-binding contract on the `chooser.md` overlay (and
ideally surface it as a hint from `chooser.add_column` / `chooser.create`
when a property-driven column is added with no resolvable binding):

- `chooser.create` should call out `contextObjectType` as required for
  any chooser whose columns read context properties (i.e. essentially
  all input columns), not an optional field.
- `chooser.add_column(kind=float|bool|object|enum)` docs should state
  that a `propertyBinding` (a property path on the context class) is
  required for the column to bind, and that omitting it produces a
  `No Property Bound` diagnostic at `chooser.compile`.
- A short "build order" note: set `contextObjectType` first, then add
  each column with its `propertyBinding`, then `add_row` / `set_cell` /
  `set_result`, then `compile` — and that adding columns *after* rows is
  not the documented order.

## Evidence

From this task's friction note (task `chooser.create`, ~31 MCP calls,
focus `chooser`): the first build did `create` (no `contextObjectType`)
→ `add_column float` → `add_column bool` → 3 rows → 6 cells → 3 results →
`chooser.compile`, which returned **2x `No Property Bound`** error
diagnostics ("column 0 + 1 because ContextData empty"). The agent then
had to read the engine source `ChooserPropertyAccess.cpp` to learn the
cause, `asset.delete` the entire `CH_DisplayPropSelector` table, and
**rebuild the whole chooser from scratch** with
`contextObjectType=StaticMeshComponent` and
`propertyBinding=MinDrawDistance` / `bVisible` — semantically forced
mappings ("no stock class cleanly exposes 'viewer distance'/'is hero'").
Quoting the note: *"the wiki never says columns need a
context+propertyBinding to compile clean, so I had to read the engine
ChooserPropertyAccess.cpp to learn the cause, then delete and rebuild the
whole chooser."* The second build (with bindings) compiled with
`diagnostics:[]`. Net process cost: one wasted full build + one engine
source dive + one asset delete, all avoidable with a one-paragraph wiki
note.

This is a **docs/discoverability** gap, distinct from the read-back
tool-bug filed as
[B-instanced-struct-export-opaque](B-instanced-struct-export-opaque.md):
that ticket is about the authored state being invisible on read-back;
this one is about the *authoring contract* (columns need bindings) being
undocumented up front, which is what caused the delete-and-rebuild.

**Fix:** Expand `docs/wiki-src/chooser.md` with the column-binding
contract and build-order note above (downstream wiki process). Optional
follow-up: have `chooser.add_column` return a non-fatal hint when a
property-driven column is added with no `propertyBinding` or no context
class set.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of task
  `chooser.create` (~31 calls): `chooser.create` with no
  `contextObjectType` plus `add_column(float)`/`add_column(bool)` with no
  `propertyBinding` both succeed silently, but the first `chooser.compile`
  returned 2x `No Property Bound` error diagnostics after the full table
  was already built. Recovery required reading engine
  `ChooserPropertyAccess.cpp`, `asset.delete` of the whole table, and a
  from-scratch rebuild with `contextObjectType=StaticMeshComponent` +
  `propertyBinding=MinDrawDistance`/`bVisible`. The `docs/wiki-src/chooser.md`
  overlay is a one-line stub that never states input columns need a
  context object + per-column `propertyBinding` to compile clean, nor the
  required build order. Proposes documenting the contract on the overlay
  page (and an optional non-fatal hint from `add_column`/`create`).
  Distinct PROCESS/docs angle from the read-back bug
  B-instanced-struct-export-opaque.
- `#2-fix` `IN-REVIEW` developer — Implemented both the core docs fix and the
  ticket's named optional follow-up. (1) Expanded the `docs/wiki-src/chooser.md`
  overlay from a one-line stub into the full column→context→propertyBinding
  contract: a `## The column → context → propertyBinding contract` section (input
  columns read a context property; `contextObjectType` is effectively mandatory;
  each input column needs a `propertyBinding`; the gap only surfaces at compile as
  `No Property Bound`; no retrofit verb exists so recovery = delete + rebuild), a
  `## Build order` section (create-with-context → add_column-with-binding → rows →
  cells → results → compile), and `### chooser.create` / `### chooser.add_column` /
  `### chooser.compile` per-method H3 sections (render only on direct `call()`, zero
  root-index cost). (2) Added a non-fatal `hints` array to the `chooser.add_column`
  response when a property-driven column (bool/float/enum/object) is added with no
  context class on the table and/or no `propertyBinding`, naming the `No Property
  Bound` failure it will hit at compile; `randomize` (no input property) emits no
  hint and the column is still added (heads-up, not an error). Also tightened the
  `contextObjectType` / `propertyBinding` RPC param descriptions to state they are
  required for property-driven columns rather than merely "optional". Files:
  `Source/EditorAutomationRpcGateway/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp`
  (new `IsPropertyDrivenKind` / `BuildColumnBindingHint` helpers + wiring in
  `HandleAddColumn` + param-spec text), `Docs/wiki-src/chooser.md`. Test:
  `EditorAutomationRpcGateway.chooser.AddColumnUnboundEmitsHint` in
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestChooserAuthoringHandlers.cpp`
  asserts an unbound bool column returns a hint mentioning `No Property Bound`,
  while a context+binding column and a randomize column return no hint — it fails if
  the hint code is reverted.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. Path(s) here that move by more than the prefix in this ticket, taken from the plugin's rename history rather than the prefix rule: `Source/EditorAutomationRpcGateway/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp` → `Source/PinWrightChooser/Private/Handlers/Chooser/ChooserAuthoringHandler.cpp`; `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestChooserAuthoringHandlers.cpp` → `Source/PinWrightChooser/Private/Tests/Assets/TestChooserAuthoringHandlers.cpp`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
