---
id: E-widget-add-then-rename-discoverability
title: "Building a named widget tree from scratch: agents do add-then-rename instead of naming at add (or one import_xml)"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, widget-tree, add, rename, import_xml]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# Building a named widget tree from scratch: add-then-rename instead of naming at add

When an agent builds a widget tree from scratch and already knows the final,
intention-revealing widget names, the natural-but-redundant pattern is to call
`widget.add` (which yields auto-generated names like `Button_0`, `TextBlock_0`),
then issue one `widget.rename_widget` per widget to fix each name. This doubles
the mutation-call count for the naming portion of the task.

`widget.add` already accepts the new child's name (`name` / canonical
`widgetName`, per `E-widget-remove-widget-param-name`), so the rename pass is
entirely avoidable: name each widget at add time. Alternatively, for a fully
known tree, `widget.import_xml` builds the whole named tree in **one** call
(the overlay already calls it "substantially faster than chained `widget.add`
calls" for "large rewrites where you know the full final state").

Neither shortcut is surfaced at the point of decision. The `wiki-src/widget.md`
overlay documents `widget.import_xml` as the single-call tree builder but does
**not** mention that `widget.add` takes a name, nor does it steer the
from-scratch-with-known-names case away from add-then-rename. `rename_widget` is
the right tool for renaming an *existing* widget (e.g. cleaning up an inherited
or imported tree); it is the wrong tool when you control the name at creation.

This is a discovery/process gap, not a tool bug — every call in the observed
task succeeded first try. But the call shape was 5 `widget.add` (default names)
+ 4 `widget.rename_widget` + a re-`describe`, where the same result is one
`widget.import_xml` (all four leaf widgets named inline) plus a final
`describe`, or 5 named `widget.add` calls with no rename pass at all.

**Fix (downstream, wiki only):** In `docs/wiki-src/widget.md`, near the
`widget.add` / `widget.import_xml` material, add a short "naming widgets" note:
(1) `widget.add` accepts `name`/`widgetName` — pass the final name at add time
so no rename is needed; (2) for a fully known tree, prefer one
`widget.import_xml` with explicit `name=` attributes; (3) reserve
`widget.rename_widget` for renaming widgets you did not create (inherited /
imported / previously-auto-named trees).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a `widget.rename_widget`
  task ("build main menu HUD from scratch with descriptive names"). self_report:
  added a VerticalBox + TextBlock + 3 Buttons "with default-style names", then
  renamed all four (TitleLabel/PlayButton/OptionsButton/QuitButton). friction
  note said "none — every call succeeded first try, no retries or python
  fallback", yet the call-log shows the redundant workaround shape: 5 `widget.add`
  + 4 `widget.rename_widget` + an extra `widget.describe` for the from-scratch
  named-tree intent. `widget.add` already takes `name`/`widgetName`
  (per `E-widget-remove-widget-param-name`, DONE) and `widget.import_xml` builds
  a named tree in one call — neither is steered toward in `wiki-src/widget.md`,
  so agents fall into add-then-rename. Distinct from `E-xml-unnamed-widgets`
  (warn when `import_xml` widgets *lack* names) and `F-widget-add-placement`
  (ordered placement) — this is the discovery gap that the rename pass is
  avoidable when you control names at creation.
