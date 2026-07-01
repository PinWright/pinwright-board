---
id: E-text-contract-runtime-vs-asset-asymmetry
title: "ui.md doesn't warn that asset-side text (widget.set/property.set) requires NSLOCTEXT while runtime ui.set_widget_text takes a plain string"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, ui, widget-set, set-widget-text, ftext, localization, nsloctext]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Text-setting verbs split their contract across namespaces, and the docs don't cross-warn

A HUD-prototyping task naturally uses **two** text-setting verbs in one sitting, and they have **opposite** input contracts for the same conceptual value (a text-block's text):

- `ui.set_widget_text` (runtime, live instance) — accepts a **plain string** (`"WAVE 3 - 1200 pts"`). Works first try.
- `widget.set` / `property.set` (asset-side, persisted `FText`) — **rejects** a plain string with `INVALID_PROPERTY` / `INVALID_TEXT_LOCALIZATION_IDENTITY` and demands `NSLOCTEXT("ns","key","src")` (the persisted-localization-identity policy from the DONE `F-require-ftext-localization-identity`).

The asset-side requirement is **working as designed** and is well documented on the **authoring** pages (`widget.md` "Text and localization", `property.md`, `widget.failure-modes.md`). The gap is that the **runtime** page `ui.md` — the page you read when you do this task — documents `ui.set_widget_text` as "targets the live runtime instance; asset-side text changes belong on `property.set` / the widget CDO" but **never warns the caller that the two verbs disagree on plain-string input**. A reader who authored a HUD with plain strings via `widget.set` (or expects symmetry with the runtime verb they just used) hits the rejection cold. This is pure discoverability friction, not a tool defect.

## Evidence (this task)

Friction note (verbatim): *"(2) `widget.set` rejected a plain-string Text on the asset CDO with INVALID_PROPERTY, requiring NSLOCTEXT wrapping (documented behavior, minor) — note the runtime `ui.set_widget_text` correctly took a plain string."*

Call log shows the misuse-then-correct cycle on the asset side: call #25 `widget.set StatusText plain-string Text` → `is_error:true [INVALID_PROPERTY] Persisted FText values require a non-empty namespace and key; pass NSLOCTEXT("Namespace","Key","Source")...`; call #26 `widget.set StatusText NSLOCTEXT text+font+slot` → ok. Later, the runtime call (#37) `ui.set_widget_text StatusText = 'WAVE 3 - 1200 pts'` took the plain string with no complaint — exactly the asymmetry the reporter flagged. One wasted call + one error, every HUD authoring+driving session that sets text both ways.

## What it should do

Docs-only. On the `docs/wiki-src/ui.md` overlay page, in the `ui.set_widget_text` section (which already contrasts live vs asset and points to `property.set`), add one line: the runtime verb takes a **plain string**, but the asset-side `widget.set` / `property.set` require a **localization identity** (`NSLOCTEXT("ns","key","src")`) for new persisted `FText` values — link to `widget.md` "Text and localization". Optionally mirror a one-line back-reference from `widget.md`'s text section noting the runtime verb is plain-string. No code change; the rejection itself is correct policy.

## Related

- `F-require-ftext-localization-identity` (DONE) — the policy that makes asset-side plain strings rejected; this ticket only asks the runtime-side docs to warn about the split, not to change the policy.
- `F-widget-set-ftext-nsloctext-parse` (DONE) — added the NSLOCTEXT parser the asset side requires.
- `B-variable-category-ftext-localization-error` (IN-REVIEW) — a different value class (editor-only category label) wrongly swept into the same gate.
- Distinct from `B-screenshot-omits-umg-overlay` (the judge-filed outcome bug from this same task) — that's a capture defect; this is a docs/contract-asymmetry friction.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a HUD prototype task (author WBP_ActionHUD, drive it live in PIE). Caller hit a misuse-then-correct cycle: asset-side `widget.set` of a TextBlock `Text` (call #25) rejected the plain string with `[INVALID_PROPERTY] Persisted FText values require a non-empty namespace and key`, then succeeded with NSLOCTEXT (call #26); the runtime `ui.set_widget_text` (call #37) took the same kind of value as a plain string. The asset rejection is correct policy (`F-require-ftext-localization-identity`) and is documented on `widget.md`/`property.md`, but the runtime page `ui.md`'s `ui.set_widget_text` section — read when doing this task — never warns the two verbs disagree on plain-string input. Proposing a one-line cross-warning on `docs/wiki-src/ui.md` (and an optional back-link from `widget.md`). Docs-only; no code change.
