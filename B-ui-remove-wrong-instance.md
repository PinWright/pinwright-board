---
id: B-ui-remove-wrong-instance
title: "`ui.remove_widget_from_viewport` removes the first same-named UUserWidget from any world instead of the requested viewport instance"
status: OPEN
severity: High
category: bug
tags: [ui, widget, viewport, world-selection, wrong-target]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Named viewport removal can target a preview or another PIE world

## What happens

The empty-key branch scopes enumeration to the game viewport's world
(`UiHandler.cpp:362-377`). The named branch does not: it walks the global
`TObjectIterator<UUserWidget>`, accepts the first object whose short name matches
and whose world is merely non-null, calls `RemoveFromParent`, and stops
(`:383-400`). It does not require the game viewport world, the intended PIE
instance, or even `IsInViewport()`.

Editor previews and multi-client PIE can contain same-named widget instances. Their
global iteration order is not a target identity, so the handler can remove an
unpainted/foreign widget and report success while the requested viewport widget
remains.

## Why it matters

This is a cross-world destructive write followed by a false success response.
Severity is High because the removal is reversible but silently targets foreign
caller state.

## What should happen

Use the PIE-first runtime widget collector/selector already used by the neighboring
UI handlers. Require viewport membership and explicit world/player identity, refuse
ambiguous names, and return the owning world, player and actual object path. Verify
the selected widget is no longer attached before success.

## Workaround

Keep live widget names unique across open previews and PIE clients. The empty-key
form is correctly world-scoped but removes all viewport widgets, so use it only when
that destructive behavior is acceptable.

## Related

- `B-set-widget-text-hits-wrong-instance` — same wrong-instance family on setters.
- `F-widget-describe-live-root-disambiguation` — documents live-root identity but
  does not fix this remover.

## History
- `#1-source-scan-global-widget` `OPEN` reporter — Source-only scan contrasted the game-viewport-scoped empty-key branch with the named branch's global first-name match and found no world, player, viewport, or ambiguity check. No build, test, editor, MCP call, or plugin edit was performed.
