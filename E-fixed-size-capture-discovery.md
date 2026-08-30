---
id: E-fixed-size-capture-discovery
title: "editor.screenshot entry doesn't redirect fixed-size capture intent to render.capture_open_level"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, editor, render, screenshot, viewport, resolution, discovery]
---

# editor.screenshot has no breadcrumb to render.capture_open_level for an exact-size capture

A caller whose intent is "capture the level-editor viewport at an exact target
size" (press kit / hero shot at a fixed QHD output) reaches `editor.screenshot`
as the obvious capture verb. But `editor.screenshot` captures the level viewport
at its **native** size with no fixed-resolution control (see
`F-editor-viewport-screenshot`, DONE — the native-size fallback capability), so
the capture lands at the viewport's small native resolution regardless of the
requested size, and nothing on its docs entry points the caller at the verb that
*does* take a size.

The method that actually does what the intent wants already exists:
**`render.capture_open_level`** — documented in `docs/wiki-src/render.md` as
*"exact-size PNG captures from ... the active Level Editor viewport"* with
caller-provided camera/projection/**resolution**, and listed in the
`visual-review` capture-surface table for the "Open Level Editor viewport" row.
But the `editor.md` `editor.screenshot` entry doesn't cross-reference it:

- `docs/wiki-src/editor.md` lists `editor.screenshot` only on the "Stats /
  capture" bullet, with no `### editor.screenshot` section. It does not say the
  level-viewport fallback is *native size only* and does not redirect fixed-size
  captures to `render.capture_open_level`.
- The only place the right verb is reachable is `visual-review.md` / `render.md` —
  pages a caller has no reason to open when `editor.screenshot` appears to cover
  the task.

This is the *discovery* angle on the *resolution* axis, distinct from
`E-viewport-screenshot-name-discovery` (OPEN, the *name* axis:
`viewport.screenshot`→`editor.screenshot`). Both touch the `editor.screenshot`
entry, but with complementary one-liners (this one routes a fixed-size intent to
`render.capture_open_level`; that one routes a `viewport.*` name guess to
`editor.screenshot`).

> **Note — the misc half of the original report is already handled.** This ticket
> was originally filed proposing *two* breadcrumbs: one on
> `misc.set_viewport_resolution` and one on `editor.screenshot`. The
> `misc.set_viewport_resolution` half is now obsolete: its sibling
> `B-set-viewport-resolution-noop` (IN-REVIEW) replaced that method's silent
> input-echoing no-op with a hard `SendError("NOT_IMPLEMENTED", …)` whose message
> *already* names `render.capture_open_level {width, height}` and the `HighResShot`
> fallback (`MiscHandler.cpp:255,261-265`), and rewrote the `REGISTER_RPC_HANDLER`
> description (the only contract surface, since `misc.md` has no
> `set_viewport_resolution` entry) to match. That delivers the misc-side breadcrumb
> on the live-error surface the workflow actually hits — better than a `misc.md`
> doc note — and removes the misleading `2560x1440` echo entirely, so the original
> "~15-RPC dead-end" framing no longer applies. Only the `editor.screenshot`
> redirect below is still genuinely missing.

## What it should do

Docs-only (cheapest, no behavior change). In `docs/wiki-src/editor.md`, on the
`editor.screenshot` entry, note that outside PIE it captures the level viewport at
its **native** size (no fixed-resolution control here) and redirect fixed-size
needs to `render.capture_open_level` (caller-provided `width`/`height`); see
`visual-review` for choosing a capture surface.

## Evidence

From the `misc.set_viewport_resolution` struggle audit (namespace `misc`, outcome
`tool_bug`). Friction note, verbatim: *"the core press-target requirement is unmet
— misc.set_viewport_resolution uses r.SetRes which targets the game/PIE window,
while editor.screenshot fell back to the small (957x231) level-editor viewport at
'native resolution' with no PIE world, so the two calls are disconnected ... which
is itself a discoverability/capability gap for the stated press-kit workflow."*
The right verb exists and is documented: `docs/wiki-src/render.md` ("exact-size PNG
captures from ... the active Level Editor viewport", `### render.capture_open_level`)
and the `visual-review.md` capture-surface table ("Open Level Editor viewport |
`render.capture_open_level`"); `editor.md` does not cross-reference it.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the misc.set_viewport_resolution
  struggle audit (outcome tool_bug; bug itself filed as `B-set-viewport-resolution-noop`).
  PROCESS angle: a fixed-size level-viewport capture intent chains the obvious
  `misc.set_viewport_resolution` + `editor.screenshot` pair (disconnected; lands at
  native 957x231) and never discovers `render.capture_open_level`, the documented
  exact-size open-level capture verb, because neither `misc.md` nor `editor.md`
  cross-references it (only `render.md` / `visual-review.md` do). The misleading
  2560x1440 echo let the workflow commit ~15 RPCs before the size check failed at
  the final `system.job_status` readback. Proposed: docs-only — add a
  `set_viewport_resolution`→`render.capture_open_level` note on `misc.md` and a
  native-size + redirect note on the `editor.screenshot` entry in `editor.md`.
  Tagged `docs`; overlays to edit are `docs/wiki-src/misc.md` and
  `docs/wiki-src/editor.md`. Distinct from `B-set-viewport-resolution-noop` (the
  bug) and `E-viewport-screenshot-name-discovery` (the screenshot-name breadcrumb).
- `#2-reword-rescope-to-editor-screenshot` `IN-REVIEW` developer — Reworded and
  rescoped (severity Medium→Low). The misc half of the original two-part proposal
  is already delivered by the sibling `B-set-viewport-resolution-noop` (IN-REVIEW):
  `MiscHandler.cpp:255,261-265` now returns `NOT_IMPLEMENTED` with a message that
  names `render.capture_open_level {width, height}` and the `HighResShot` fallback,
  and the registry description (the only contract surface — `misc.md` has no
  `set_viewport_resolution` entry) was rewritten to match. That breadcrumb fires on
  the live-error surface the workflow actually hits and removes the misleading
  `2560x1440` echo, so the original "~15-RPC dead-end" / silent-echo premise is
  stale and was dropped from the body. The remaining genuine, unowned gap is the
  `editor.screenshot` redirect on the *resolution* axis (distinct from the *name*
  axis of `E-viewport-screenshot-name-discovery`, OPEN, which routes the
  `viewport.screenshot` guess to `editor.screenshot`). Implemented docs-only: added
  a `### editor.screenshot` H3 overlay section to `docs/wiki-src/editor.md` stating
  the outside-PIE fallback captures the level viewport at its **native** resolution
  with no fixed-size control, and redirecting exact-size captures to
  `render.capture_open_level` (caller-provided `width`/`height`; see `visual-review`),
  plus a parenthetical that `misc.set_viewport_resolution` returns NOT_IMPLEMENTED.
  The H3 surfaces only on `call("editor.screenshot")` (no namespace-page or root
  bloat). Regression test: `FWikiHandlerScreenshotRedirectsFixedSizeTest` in
  `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp`
  renders `editor.screenshot` through the production `WikiHandler::RenderPage` and
  asserts the page contains `render.capture_open_level` and `native` — it fails if
  the overlay section is reverted (the auto summary alone never names
  `render.capture_open_level`). Files: `docs/wiki-src/editor.md`,
  `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp`. Not
  compiled/run here (later phase verifies).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
