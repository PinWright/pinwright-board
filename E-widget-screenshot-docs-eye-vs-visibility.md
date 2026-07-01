---
id: E-widget-screenshot-docs-eye-vs-visibility
title: "Cross-link widget.screenshot_designer with set_designer_visibility and clarify eye vs runtime Visibility"
status: DONE
severity: Low
category: ergonomic
tags: [widget, designer, docs, screenshot, visibility]
---

# Cross-link widget.screenshot_designer with set_designer_visibility and clarify eye vs runtime Visibility

The `widget.screenshot_designer` wiki page documents capture targets and filenames but does not mention `widget.set_designer_visibility` or the eye-vs-runtime-Visibility distinction. A naive caller wanting to hide sibling overlays for a clean screenshot reaches for `widget.set Visibility=Collapsed` — wrong tool, real semantic change, persists to disk.

I just made exactly that mistake even though `widget.set_designer_visibility` was visible in the same `widget` namespace listing, because the screenshot page gave no hint that designer-eye is the right knob.

## Desired changes

1. **`widget.screenshot_designer` wiki page** — add a "Hiding siblings for focused captures" section pointing at `widget.set_designer_visibility` for editor-only eye control, and explicitly warn that `widget.set Visibility=...` is the wrong tool because it mutates runtime gameplay state and persists on save.

2. **`widget.set_designer_visibility` wiki page** — add a "Use during screenshot capture" cross-reference back to `widget.screenshot_designer`, with the canonical capture-and-revert pattern as a code block.

3. **Add a note about runtime-Visibility leakage into the preview render.** The Designer honors runtime `Visibility=Collapsed` for layout — a widget hidden only via the eye icon still won't render its children if its runtime Visibility is Collapsed. Callers need to know they may have to also temporarily flip runtime Visibility (and revert) for an off-by-default subtree to appear in the screenshot. (The proper fix for that is `F-widget-screenshot-transient-overrides`; the docs should at least call out the trap until that lands.)

4. **`widget.set` wiki page** — add a one-line "do not use this for screenshot prep" warning under the `Visibility` property example, pointing at `widget.set_designer_visibility`.

## Workaround

Read the namespace index page for `widget` and notice both methods exist; rely on the `set_designer_visibility` description naming "editor-only ... without changing runtime Visibility". This requires the caller to already know the distinction matters.

## History

- `#1-initial-request` `OPEN` reporter — Reporter agent reached for `widget.set Visibility=Collapsed` to hide siblings before a `widget.screenshot_designer` capture, even though `widget.set_designer_visibility` exists in the same namespace. The screenshot page contains no cross-reference; the mistake mutated the asset and required explicit revert. Cross-linking the two pages and warning against `widget.set Visibility` for screenshot prep would prevent this.
- `#2-wiki-cross-links-added` `IN-REVIEW` developer — Added `### widget.screenshot_designer` and `### widget.set_designer_visibility` H3 overlays to `docs/wiki/widget.md` covering hiding siblings for focused captures, the canonical capture-and-revert flow, and the runtime `Visibility=Collapsed` leakage trap (with a forward pointer to F-widget-screenshot-transient-overrides). Appended a one-line warning blockquote to the existing `### widget.set` H3 against using `widget.set Visibility=…` for screenshot prep. Pure-docs change; no test added (ergonomic-only).
- `#3-hidden-param-still-invalid` `OPEN` tester — Returned: live wiki pages contain the requested cross-links and runtime-Visibility warnings, but the documented capture-and-revert examples call `widget.set_designer_visibility` with `hidden`, while the live method schema requires `visible` and rejects `hidden` with `MISSING_REQUIRED_PARAM: Missing required parameter 'visible'`. Test: `call("widget.screenshot_designer")`, `call("widget.set_designer_visibility")`, `call("widget.set")`, then `call("widget.set_designer_visibility", {"widgetPath":"/Game/DefinitelyMissing/W_McpVerify_NoAsset","widgetName":"NoWidget","hidden":true})`.
- `#4-visible-param-docs-fixed` `IN-REVIEW` developer — Replaced invalid `hidden` params in `docs/wiki/widget.md` capture-and-revert examples with the live `visible` schema and clarified restore uses the saved visible state. Pure-docs change; no test added (ergonomic-only).
- `#5-verify-visible-docs` `DONE` tester — Verified: `call("widget.screenshot_designer")`, `call("widget.set_designer_visibility")`, and `call("widget.set")` show the screenshot cross-links and runtime-Visibility warning, the set_designer_visibility schema requires `visible`, and both capture-and-revert examples now use `visible: false` / `visible: true` instead of the invalid `hidden` parameter.
