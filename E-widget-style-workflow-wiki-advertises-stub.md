---
id: E-widget-style-workflow-wiki-advertises-stub
title: "widget namespace prelude advertises 'configure styles' but no page documents the real styling path (create_style adds vars; apply_style returns NOT_IMPLEMENTED; the working route is widget.set WidgetStyle / a BPIR binding) — agent had to source-dive"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [widget, wiki, docs, discovery, apply-style, create-style, stub]
---

# widget styling workflow is undocumented and the prelude over-promises it

The `widget` namespace overview (`docs/wiki-src/widget.md:3`) opens by listing
"create widget blueprints, edit their widget tree, **configure styles**, attach
event bindings, drive widget animations" as a flat capability claim. A reader
scanning the prelude reads an unqualified "configure styles" promise — but **no
page in the widget wiki documents the real styling path**: that `create_style`
adds style member variables named `<styleName>_<type>`, that `widget.apply_style`
does *not* apply anything (it now returns `NOT_IMPLEMENTED` after the
`B-widget-apply-style-silent-noop` fix), and that the way to actually style a
widget is to set its `WidgetStyle` directly via `widget.set` (or wire a runtime
Blueprint property binding via BPIR).

Concretely, the discovery surface fails the styling task two ways:

1. **`widget.md` prose has zero coverage of styling.** It documents the
   XML round-trip, event bindings, asset-path aliases, describe/export modes,
   and screenshots — but never `create_style`, the `<styleName>_<type>`
   variable-naming convention it produces (e.g. `_ButtonStyle`, `_Font`,
   `_Brush`, `_ProgressStyle`), the fact that `apply_style` is `NOT_IMPLEMENTED`,
   or the working `widget.set WidgetStyle` / BPIR-binding alternative.
2. **`widget.failure-modes.md` — the page that literally says "Consult this
   when something visible (or invisible) doesn't match what you authored" — has
   no symptom→cause→fix row for the styling task.** Its table covers empty
   buttons, layout, fonts, orphan widgets, NSLOCTEXT literals, etc., but a
   caller who reached for `create_style`/`apply_style` to style a widget and got
   nowhere finds no row pointing at the working route.

So wiki navigation steered the caller toward the styling verbs with no warning
that `apply_style` applies nothing and no pointer at what does. The only way to
learn the working path was a **C++ source-dive** (`WidgetStyleHandler.cpp`) —
exactly what happened in this task.

## Distinct from the bug ticket

`B-widget-apply-style-silent-noop` (IN-REVIEW) was the **C++ honesty fix** —
`apply_style` no longer fake-succeeds; it now returns `SendError("NOT_IMPLEMENTED",
…)` (`WidgetStyleHandler.cpp:209-211`) and its registration summary points at the
real route. That fix changed the *runtime* honesty but did **not** touch the
discovery surface: `widget.md:3` still over-promises "configure styles", there is
still no documented styling workflow, and `widget.failure-modes.md` still has no
styling row. This ticket is the **docs-overlay companion** the C++ ticket
explicitly deferred — it survives the fail-loud fix (a NOT_IMPLEMENTED return
still wants a documented working alternative + a failure-mode row).

This is the same shape as the accepted overlay precedents
`E-texture-create-wiki-advertises-stub`,
`E-ik-rig-family-wiki-advertises-compiled-out-workflow`,
`E-level-structure-wp-wiki-advertises-dead-end`, and
`E-set-transition-rules-wiki-overstates-rule-authoring`: the namespace overview
advertises a capability the runtime does not deliver; the fix is a
`docs/wiki-src/` overlay edit (downstream wiki process, not this audit).

## What the wiki should do

In `docs/wiki-src/widget.md`:
- Qualify the prelude so "configure styles" is not an unqualified working-capability
  claim (`apply_style` does not apply anything).
- Add a namespace-page-visible `## ` section documenting the real styling path:
  `create_style` adds style member variables named `<styleName>_<type>` (e.g.
  `PrimaryButtonStyle_ButtonStyle`); `apply_style` returns `NOT_IMPLEMENTED`
  (applies nothing); and the working alternative — set the widget's
  `WidgetStyle` directly via `widget.set` with an `FButtonStyle` ExportText
  literal, or wire a property binding via BPIR — with the instruction to
  **verify with `widget.describe` / `export_xml`**. Keep it below the prelude so
  it renders on the namespace page but not the root index.

In `docs/wiki-src/widget.failure-modes.md`:
- Add a symptom→cause→fix row: "Tried to style a widget with `create_style` +
  `apply_style` and nothing changed" → "`apply_style` is `NOT_IMPLEMENTED` (it
  applies no style and creates no binding); `create_style` only adds
  `<styleName>_<type>` member variables" → "set `WidgetStyle` directly via
  `widget.set`, or wire a property binding via BPIR; verify with `widget.describe`."

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs audit of task focus `widget.apply_style` (story: author WBP_SettingsMenu, define one PrimaryButtonStyle, apply it to 3 buttons, read back). Friction note: "I had to read the plugin C++ (WidgetStyleHandler.cpp) to confirm apply_style applies nothing; no in-tool path makes the style show in describe/export_xml." Call-log: 3× `widget.apply_style` each returned `success:true` + `styleFound:false` + "requires runtime binding setup", then `widget.describe`+`export_xml`+a second targeted `describe` readback all confirmed nothing applied — a source-dive resolved it. Root discovery cause: `docs/wiki-src/widget.md:3` advertises "configure styles" but the page documents NO `create_style`/`apply_style` workflow and carries NO stub caveat; `widget.failure-modes.md` (the "something invisible doesn't match what you authored" page) has no row for this. Distinct from `B-widget-apply-style-silent-noop` (the per-finding judge's C++ honesty bug) — this is the docs advertising the trap and survives even a fail-loud fix. Same accepted overlay shape as `E-texture-create-wiki-advertises-stub` / `E-ik-rig-family-wiki-advertises-compiled-out-workflow` / `E-level-structure-wp-wiki-advertises-dead-end` / `E-set-transition-rules-wiki-overstates-rule-authoring`. Dedup: ripgrep over board for `styl`/`apply.style`/`create_style`/`WidgetStyle` and `ls *widget*styl*` found no widget-styling docs ticket; only `B-widget-apply-style-silent-noop` (the C++ bug) exists. Names pages to edit: `docs/wiki-src/widget.md`, `docs/wiki-src/widget.failure-modes.md`.
- `#2-retriage` `OPEN` triage — Low→Medium: wiki advertises a capability backed by a silent no-op stub, steering agents into a source-dive; niche styling path.
- `#3-reword-and-overlay` `IN-REVIEW` developer — Reworded down to current reality: the sibling C++ ticket `B-widget-apply-style-silent-noop`'s fail-loud fix already landed, so `apply_style` now returns `SendError("NOT_IMPLEMENTED", …)` (`WidgetStyleHandler.cpp:209-211`) and its registration summary (`:192`) carries the caveat + workaround that auto-renders into the `## Methods` index and per-method page. Dropped the stale `success:true`/"silent no-op stub" framing and the now-false finding #3 (per-method page already caveated); the live, surviving gaps are (a) the `widget.md:3` prelude still over-promising "configure styles" and (b) `widget.failure-modes.md` having no styling row. Edited `Docs/wiki-src/widget.md` — softened the prelude ("configure styles"→"set widget properties") and added a namespace-page-visible `## Styling` section documenting the real path: `create_style` adds `<styleName>_<type>` member vars (e.g. `PrimaryButtonStyle_ButtonStyle`, `_ProgressStyle`), `apply_style` is `NOT_IMPLEMENTED` (applies nothing / creates no binding), and the working route is set `WidgetStyle` directly via `widget.set` (FButtonStyle ExportText) or a BPIR property binding, verify with `widget.describe`/`export_xml`. Edited `Docs/wiki-src/widget.failure-modes.md` — added a symptom→cause→fix row for the styling task pointing at the same working route. Regression test: `FWikiHandlerWidgetDocumentsStylingPathTest` in `Tests/Infra/TestWikiHandler.cpp` renders the `widget` namespace page via `WikiHandler::RenderPage` and asserts the overlay-exclusive markers `PrimaryButtonStyle_ButtonStyle`, `_ProgressStyle`, and "creates no binding" (none appear in any handler registration summary, so it fails iff the `## Styling` overlay section is reverted). No C++ behavior changed (docs overlay + test only). Removed the `claimedBy`/`claimedAt` lease.
