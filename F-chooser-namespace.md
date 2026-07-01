---
id: F-chooser-namespace
title: "Chooser first-slice asset/row/column authoring (chooser.*)"
status: DONE
severity: High
category: feature
tags: [chooser, animation, authoring, asset-creation, missing-namespace]
---

# Chooser first-slice asset/row/column authoring (`chooser.*`)

Zero RPCs in the plugin touch `UChooserTable`. Grep across
`Source/EditorAutomationRpcGateway/Private/Handlers/` returns no hits
for `chooser` or `UChooserTable`, and the generated method reference
(`docs/rpc-method-reference.generated.md`) contains no `chooser.*`
namespace. The Chooser editor plugin
(`C:/UE_5.6/Engine/Plugins/Chooser/`) ships as a stable, non-experimental
UE plugin with first-class editor tooling, and is widely used to drive
animation selection (montages, sequences, AnimNext graphs), gameplay
table dispatch, and data-driven asset reference resolution. AnimNext's
own AnimNextChooser plugin builds on it. Agents cannot currently author
chooser tables imperatively — they must hand-edit the asset in the
editor or compose it by direct property writes on the loaded
`UChooserTable`, which bypasses the column/row API contracts and
`PostEditChange` triggers.

The local UE 5.6 Chooser column-parameter surface includes object,
float, bool, enum, gameplay tag, struct, and randomize families, but
the first implementation slice should expose only the stable contracts
needed for table creation and basic row population. UE 5.6 does not
ship a first-class `IntColumn` / `IChooserParameterInt`, so an MCP int
column must not be faked with another storage type.

**Proposal:** Add a narrowed `chooser.*` namespace mirroring the typed
authoring style used by `niagara.authoring.*` and
`animation.authoring.*`. First-slice methods:

- `chooser.create(path, contextObjectType?, resultType?, save?)` —
  create a new `UChooserTable` asset; optionally sets a context object
  class and result type.
- `chooser.add_column(path, kind, propertyBinding?, save?)` — appends
  a concrete `FInstancedStruct` column where `kind` is one of `bool`,
  `float`, `enum`, `object`, or `randomize`.
- `chooser.add_row(path, save?)` — appends a row, initializes
  `ResultsStructs`, calls each column's row sizing API, and returns
  the row index.
- `chooser.set_cell(path, row, column, value, save?)` — typed setter
  dispatched from the stored column struct.
- `chooser.set_result(path, row, resultKind, value, save?)` — binds a
  row result where `resultKind` is one of `asset`, `class`, or
  `evaluate_chooser`.
- `chooser.compile(path, save?)` — calls `Compile(true)`,
  `PostEditChange()`, marks the asset dirty, and returns diagnostics.

Explicit deferrals: `int`, `gameplay_tag`, `struct`, output-only
columns, and broader result families remain out of scope for this
ticket. They need separate engine-contract checks before becoming MCP
surface.

Column-type-specific cell setters intentionally collapse into one
`chooser.set_cell` dispatched on the stored column kind — same pattern
`niagara.authoring.set_module_input_value` uses for typed input slots
— rather than ~8 sibling RPCs (`chooser.set_cell_float`,
`chooser.set_cell_object`, ...). The column kind is already
authoritative from `add_column`; redundant verb-per-kind would add
surface without adding capability.

**Use cases blocked today:**

1. Authoring montage-selector choosers for AnimBlueprints from
   templated specs (cannot create or populate the table).
2. Test fixtures: building a small chooser to validate runtime
   selection logic via `asset.dump` + automation tests.
3. Data-driven asset reference tables for game logic (loadout pickers,
   ability variant resolution) that the team would otherwise express
   as DataTables.

**Workaround:** Hand-author the chooser asset in the editor, then
reference it via `asset.dump` for read-back. No imperative authoring
path exists.

**Note:** The proposed body referenced "mirror existing `data_table.*`
patterns for the row API" — `data_table.*` is **dump-only** in this
plugin (no `data_table.create` / `add_row` / `set_cell` exist); the
nearest authoring templates are `niagara.authoring.*` and
`animation.authoring.*`, which is what this ticket mirrors.

**Cross-ref:** Pairs with the `animation.authoring.*` family —
chooser-driven animation selection is a common consumer of the typed
node creators tracked under
[`F-anim-add-graph-node-generic`](F-anim-add-graph-node-generic.md).
Once both land, an agent can build a montage-selecting AnimBlueprint
end-to-end without leaving MCP.

## History
- `#1-no-chooser-rpcs-exist` `OPEN` reporter — Grep across
  `Source/EditorAutomationRpcGateway/Private/Handlers/` and the
  generated method reference returns zero hits for `chooser` /
  `UChooserTable`; the Chooser editor plugin ships stable in UE 5.6
  (`Engine/Plugins/Chooser/`) with ~8 column parameter kinds
  (`Object`, `Float`, `Int`, `Bool`, `Enum`, `GameplayTag`, `Struct`,
  `Randomize` per `IChooserParameter*` headers). Proposes a
  `chooser.*` namespace: `create`, `add_column`, `add_row`,
  `set_cell` (typed dispatch by stored column kind, single verb to
  avoid 8 sibling RPCs), `set_result` (typed output binding),
  `compile`. Mirrors `niagara.authoring.*` / `animation.authoring.*`
  patterns; `data_table.*` is dump-only in this plugin and provides
  no row-authoring template despite the proposed body's claim.
- `#2-first-slice-authoring` `IN-REVIEW` developer — Implemented the
  narrowed first-slice `chooser.*` namespace with `create`,
  `add_column`, `add_row`, `set_cell`, `set_result`, and `compile`;
  added Chooser module/plugin wiring and
  `FChooserAuthoringBoolClassRoundTripTest` to prove bool column cell
  storage and class result storage through production handlers while
  explicitly deferring int, gameplay tag, struct, and output-only
  columns.
- `#3-review-fix-atomic-finalization` `IN-REVIEW` developer — Addressed
  review iteration 1 by limiting `Compile(true)` to `chooser.compile`,
  resolving `chooser.set_result` targets before row mutation, validating
  optional `chooser.create` classes before package/object allocation, and
  adding failure-path handler tests for invalid create and failed result
  assignment.
- `#4-review-fix-reuse` `IN-REVIEW` developer — Reused the shared
  loose enum-token normalizer also used by EQS and delegated chooser
  package-path cleanup to `SanitizeProjectRelativePath` after the
  chooser-specific object-path and `.uasset` conversions.
- `#5-verify-chooser-reuse` `DONE` tester — Verified: `chooser.create` accepted `/Game/App/UI/Test/W_McpVerifyTemp_FChooserNamespace.uasset` and returned assetPath `/Game/App/UI/Test/W_McpVerifyTemp_FChooserNamespace`; `chooser.add_column` with `kind:"BOOL"` returned `columnIndex:0`, `columnCount:1`, and `kind:"bool"`; temp asset deleted with `asset.delete`.
