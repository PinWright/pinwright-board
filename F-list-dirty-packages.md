---
id: F-list-dirty-packages
title: "No read-only RPC to list currently-dirty packages, so a 'verify no side effects' check has to be improvised"
status: OPEN
severity: Low
category: feature
tags: [editor, dirty-package, read-only, verification, side-effects]
encounters: 2
lastSeen: 2026-06-23T10:08:23Z
---

# No non-destructive way to enumerate currently-dirty packages

There is currently no MCP method that simply **reports which packages are
dirty** without mutating editor state. The only methods that touch
package-dirtiness are destructive or save-coupled:

- `editor.save_all` returns `totalDirty`/`savedCount`, but it *saves* (writes
  to disk, may fire source-control hooks) — it cannot be used as a probe.
- `editor.quit` refuses on dirty packages (`UNSAVED_CHANGES`), but it is a
  shutdown verb, not a query.
- `python.execute` can walk dirty packages
  (`unreal.find_object`/editor-loading-library enumerations), but that is a
  full Python round-trip with stdout text instead of typed JSON, and the
  reflected API for "list every dirty UPackage" is non-obvious.

This bites the common **"prove this sequence had no side effects"** intent.
A read-only inspection/audit task whose explicit contract is *"every call is
read-only and must not dirty any package"* has no clean way to confirm that
post-condition. There is no `editor.list_dirty_packages` (or equivalent) that
returns the set of dirty package paths, so the agent must substitute a
weaker, indirect proxy for the check.

## What it should do

Add a read-only handler (e.g. `editor.list_dirty_packages` or
`system.inspect.get_dirty_packages`) that walks the loaded package set and
returns the dirty ones, with no mutation and no save:

```json
{
  "dirtyCount": 0,
  "packages": [
    { "packageName": "/Game/Maps/L_Foo", "isMap": true }
  ],
  "success": true
}
```

Implementation walks loaded `UPackage`s (e.g. `FEditorFileUtils`'
dirty-package collection, or iterate `UPackage::IsDirty()`), emitting one
row per dirty package. Zero side effects — mirrors the read-only-probe
precedent established by `F-editor-status` (DONE), which added a dedicated
zero-side-effect state probe rather than forcing callers to abuse a
destructive verb (`editor.play`) or `python.execute` boilerplate as a query.

## Evidence

From the read-only "environment audit" struggle audit (focus
`system.inspect.get_editor_settings`, namespace `system`, outcome
`tool_bug` for the separate stub bug `B-inspect-settings-stats-stub-silent-success`).
The task contract was explicitly *"Every call in this audit is read-only and
must not dirty any package."* Friction note, verbatim:

> No clean MCP "list dirty packages" method exists, so the no-side-effects
> check relied on the documented read-only guarantee plus the identical
> get_selected_actors round-trip rather than a source_control probe.

Concretely, lacking a dirty-package probe, the 13-call audit paid two
workaround calls for the post-condition: a **second** `system.inspect.get_selected_actors`
(byte-identical to the first, used purely as a "nothing changed" round-trip)
plus a `source_control` wiki-nav, instead of a single
`editor.list_dirty_packages` returning `dirtyCount: 0`.

## Distinct from

- `F-editor-status` (DONE) — that read-only probe reports PIE/editor world
  state, not package dirtiness. Same "no clean read-only state query"
  shape; orthogonal datum.
- `E-source-control-disabled-gate-undocumented` (OPEN) — that is about the
  `source_control` namespace being gated/undocumented; this is about a
  dirty-package *enumeration* existing at all (source control would not even
  see in-memory dirty-but-unsaved packages).
- `B-editor-save-all-no-completion-signal` / `E-save-all-sync-fast-path`
  (DONE) — those are about `editor.save_all`'s job/sync envelope; this asks
  for a *non-saving* read of the same dirty set.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the read-only environment-audit struggle audit (focus `system.inspect.get_editor_settings`, outcome tool_bug for the separate stub bug). Task contract was "every call is read-only and must not dirty any package," but no MCP method reports dirty packages without mutating: `editor.save_all` saves (destructive), `editor.quit` refuses (shutdown verb), `python.execute` is a stdout round-trip. Friction note verbatim: "No clean MCP 'list dirty packages' method exists, so the no-side-effects check relied on the documented read-only guarantee plus the identical get_selected_actors round-trip rather than a source_control probe." The 13-call audit spent two workaround calls (a redundant 2nd `get_selected_actors` round-trip + a `source_control` wiki-nav) substituting for a missing `editor.list_dirty_packages` that would have returned `dirtyCount: 0`. Proposed: read-only `editor.list_dirty_packages` (or `system.inspect.get_dirty_packages`) walking loaded `UPackage::IsDirty()` and returning `{ dirtyCount, packages:[{packageName,isMap}] }`, no save, no mutation — same read-only-probe precedent as `F-editor-status` (DONE). Distinct from `F-editor-status` (PIE/world state, not dirtiness), `E-source-control-disabled-gate-undocumented` (SCC gating; SCC can't see in-memory dirty), and the `save_all` job-envelope tickets (those save).
- `#2-evidence-widget-screenshot-clean-check` `OPEN` reporter — Same friction, independent task (focus `widget.screenshot_designer`, namespace `widget`, outcome clean). The story's steps 5–6 used `showOnly:[Crosshair]` and required proving the transient designer-eye override "auto-reverted (must not dirty the asset)" — a textbook "prove no side effects" post-condition, the exact intent this ticket addresses. The `widget.screenshot_designer` overrides themselves worked first try (that feature is `F-widget-screenshot-transient-overrides` DONE); the only friction the agent reported was confirming the asset stayed clean. Friction note verbatim: "the only minor note is asset.get does not surface an explicit dirty flag, so I confirmed undirtied state via the step-6 readback plus a Crosshair subtree re-describe." Lacking a dirty-package probe, the agent paid two improvised proxy calls — an `asset.get` "WBP_PlayerHUD dirty/meta check" (which has no dirty flag to read) plus a `widget.describe` "Crosshair subtree post-capture verify" — to substitute for a single `editor.list_dirty_packages` returning `dirtyCount: 0`. Note `asset.get` not exposing a dirty/isModified field is the immediate symptom here; the proposed read-only enumeration remains the clean fix (a per-asset `isDirty` on `asset.get` would also resolve the narrower case).
