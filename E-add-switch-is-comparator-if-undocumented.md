---
id: E-add-switch-is-comparator-if-undocumented
title: "material.authoring.add_switch silently makes a comparator MaterialExpressionIf; method name + selector convention undocumented"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [material, material-authoring, add-switch, if-node, docs, misuse-then-correct]
---

# `add_switch` makes a comparator `If`, not a clean bool true/false switch

`material.authoring.add_switch` is named like a boolean selector: the natural
reading is "give me a node with a True input, a False input, and a bool/condition
input, and it picks one." What it actually creates is a
`MaterialExpressionIf` — a numeric **comparator** with pins
`A`, `B`, `AGreaterThanB`, `AEqualsB`, `ALessThanB`. To use it as a
true/false switch the caller has to know an undocumented convention:

- the boolean driver (e.g. a `StaticBoolParameter`) feeds input **A**,
- a numeric reference value feeds input **B** (a 0/1 selector),
- the "true" branch goes on `AGreaterThanB` and the "false" branch on
  `ALessThanB` / `AEqualsB` (or vice-versa, depending on the selector value).

None of this is discoverable from the method. The wiki overlay
(`docs/wiki-src/material.authoring.md:32`) does carry an `If` pin-table row
(`A, B, AGreaterThanB, AEqualsB, ALessThanB`), but **nothing ties the method
name `add_switch` to that "If" node** — `add_switch` is not named anywhere in
the wiki, and the page never states the selector convention (which pin is the
bool driver, that B must carry a 0/1 reference, which result pin is "true").
A caller who never connects `add_switch` → `If` cannot find the pin table at
all.

## Why it's process friction (clean outcome, but expensive)

The task ("drop a Switch/If selector, feed true/false colors, wire the bool from
a static switch") completed cleanly, but only after a discovery + misuse-then-
correct detour:

- 3 `get_material_node_details` inspections (the If node, the static switch, and
  the Multiply) just to learn the pin names and that the bool needs a numeric
  selector.
- 2 extra `add_material_node` Constant nodes (0 and 1) + 2 `property.set` to
  manufacture the 0/1 selector values the comparator needs — capability the
  caller would not expect a "switch" to require.
- **misuse-then-correct**: the first wiring put `DayColor → If.AGreaterThanB`
  and `NightColor → If.ALessThanB/AEqualsB`; after the first `compile_material`
  the agent had to **re-wire** (`NightColor → AGreaterThanB`,
  `DayColor → AEqualsB`, `DayColor → ALessThanB`) to get the semantics right,
  then `compile_material` a second time. The branch-pin polarity was guessed
  wrong on the first pass precisely because it is undocumented.

Friction note verbatim: *"add_switch silently creates a MaterialExpressionIf
(A/B/AGreaterThanB/AEqualsB/ALessThanB pins), not a clean bool true/false
switch, and neither add_switch nor its wiki page documents the pin names or
that the bool driver must feed input A with a 0/1 selector value (I had to add
two Constant nodes + a get_material_node_details to discover this)."*

## What it should do

Ergonomic options (any one closes the gap; pick downstream):

- **Docs (minimum):** in `docs/wiki-src/material.authoring.md`, add an
  `add_switch` entry under the convenience-node section that says plainly
  "`add_switch` creates a `MaterialExpressionIf`" and links to the existing
  `If` pin-table row, plus a one-paragraph usage recipe: bool/condition → `A`,
  numeric selector → `B`, true branch → `AGreaterThanB`, false branch →
  `ALessThanB`/`AEqualsB`. Mention that driving it from a `StaticBoolParameter`
  needs a 0/1 reference on `B` and a selector value, so callers don't rediscover
  the two-Constant workaround.
- **Ergonomic (better):** have `add_switch` (or a new
  `add_static_switch_node`) emit a real bool selector node with
  `True`/`False`/`Value` pins — matching what the method name promises — so the
  comparator dance is unnecessary for the common "flip between two inputs" case.

This is filed E-/`docs` (works correctly, but the method name mismatches the
node it produces and the usage convention is undiscoverable), distinct from the
DONE `B-material-get-node-details-missing-pins-props` (that fixed *node-detail
payload contents*, not the method-name/convention gap).

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.add_switch task (focus material.authoring.add_switch, outcome clean) building M_SignBoard_DayNight day/night emissive toggle. add_switch produced a MaterialExpressionIf comparator (A/B/AGreaterThanB/AEqualsB/ALessThanB), not the bool true/false switch the name implies. Cost in the call log: 3 get_material_node_details inspections + 2 Constant add_material_node + 2 property.set to manufacture a 0/1 selector, and a misuse-then-correct re-wire (DayColor/NightColor branch pins swapped after the 1st compile, then a 2nd compile). Wiki overlay material.authoring.md:32 has an `If` pin row but never names `add_switch` or states the selector convention (bool→A, 0/1 ref→B, true→AGreaterThanB). Propose: doc the method→If mapping + usage recipe (min), or have add_switch emit a real True/False/Value bool selector (better). Medium severity — fully recoverable, but every "switch between two inputs" task pays the same discovery+rewire tax.
- `#2-docs-fix` `IN-REVIEW` developer — Implemented the docs-minimum fix (all 3 lenses voted valid; not a duplicate, no regression). Added a `### material.authoring.add_switch` H3 overlay section to `docs/wiki-src/material.authoring.md` stating plainly that `add_switch` creates a `MaterialExpressionIf` comparator (alias of `add_if`, not a bool selector), linking the existing `If` pin row, and giving the usage recipe (bool/condition→A, 0/1 numeric ref→B, true→AGreaterThanB, false→ALessThanB/AEqualsB) plus the StaticBoolParameter 0/1-reference note and the real-bool-switch alternatives (`add_static_switch_parameter` / `material.graph.add_expression` with `UMaterialExpressionStaticSwitch`, noting `add_material_node nodeType="StaticSwitch"` resolves to the parameter variant). The H3 surfaces only when an agent calls `call("material.authoring.add_switch")` directly, so it costs no namespace-page budget. Regression test: `FWikiHandlerAddSwitchDocumentsIfTest` (EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.AddSwitchDocumentsIf) in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` renders the method page via the production `WikiHandler::RenderPage` (→ `WikiOverlay::LoadMethodSection`) and asserts the page is not Not-Found and contains "MaterialExpressionIf" and "AGreaterThanB"; it fails if the overlay H3 is reverted. Docs-only fix — no plugin C++ source changed.
