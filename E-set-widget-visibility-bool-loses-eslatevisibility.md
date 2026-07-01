---
id: E-set-widget-visibility-bool-loses-eslatevisibility
title: "ui.set_widget_visibility takes a plain bool but docs promise ESlateVisibility — Collapsed/Hidden/HitTest variants silently coerced to visible:false"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ui, set_widget_visibility, eslatevisibility, docs, silent-coercion]
---

# ui.set_widget_visibility takes a plain bool but docs promise ESlateVisibility

`ui.set_widget_visibility` is documented as setting the widget's
`ESlateVisibility` state — the five-value Slate enum
`Visible / Collapsed / Hidden / HitTestInvisible / SelfHitTestInvisible`.
In practice the only honored parameter is a **plain boolean** `visible`, and
the response only echoes `visible: true|false`. A caller who passes the
documented enum string (e.g. `visible: "Collapsed"`) gets a **silent
coercion**: the string is truthy/falsy-collapsed to a bool, the response
reports `visible: true|false`, and there is no way to request `Hidden` (which
differs from `Collapsed` in layout — Hidden still occupies space) or either
HitTest variant. So three of the five real Slate visibility states are
unreachable through this method, and the two that map onto the bool are
indistinguishable in the response from the enum the caller actually asked for.

This is an ergonomic/expressiveness gap, distinct from
`B-set-widget-text-hits-wrong-instance` (which is about *which live instance*
the write lands on) and from `F-ui-common-activatable-stack` (CommonUI
activation semantics). Here the write reaches the right widget but cannot
express the requested visibility state, and the contract advertised by the
method doc does not match the param it accepts.

## What it should do

- Accept an `ESlateVisibility` string (`Visible`, `Collapsed`, `Hidden`,
  `HitTestInvisible`, `SelfHitTestInvisible`) in addition to (or instead of)
  the bool, mapping it to `UWidget::SetVisibility(ESlateVisibility)`. Keep the
  bool as a `Visible`/`Collapsed` shorthand for back-compat.
- Echo the resolved enum state in the response (e.g.
  `"visibility": "Collapsed"`), not just `visible: true|false`, so the caller
  can confirm the actual state applied.
- Reject an unrecognized enum string with a clear error listing the valid
  values, instead of silently coercing it to a bool.
- Docs (`docs/wiki-src/ui.md`, the `ui.set_widget_visibility` overlay) must
  describe the accepted parameter shape accurately — today the method doc
  implies the full enum while the schema accepts a bool.

## Evidence

From a live-HUD PIE preview task (ui namespace). The attempt agent's friction
note, verbatim:

> ui.set_widget_visibility doc claims it sets ESlateVisibility
> (Visible/Collapsed/Hidden/HitTestInvisible/SelfHitTestInvisible) but the
> param is a plain boolean and the response only echoes visible:true/false —
> 'Collapsed' is silently coerced to false and you can't request Hidden vs
> Collapsed or the HitTest variants.

Call log for the visibility step (all returned ok, but the requested enum was
not honored):
- `ui.set_widget_visibility QuestContainer visible='Collapsed'` → ok (echoed bool)
- `ui.set_widget_visibility QuestContainer visible=false` → ok
- `ui.set_widget_visibility QuestContainer visible=true` → ok

The task asked specifically to "set it Collapsed/Hidden … then bring it back to
Visible" — the Collapsed-vs-Hidden distinction the user requested cannot be
satisfied through this method.

**Workaround:** to set a non-bool ESlateVisibility state on a live widget,
there is no `ui.*` path; authoring `Visibility` on the widget BP asset
(`widget.set` / `property.set`) before PIE expresses the full enum but does not
serve the "flip a live runtime widget's visibility state" use case.

## History
- `#2-accept-eslatevisibility-enum` `IN-REVIEW` developer — `ui.set_widget_visibility` now accepts an optional `visibility` ESlateVisibility string (Visible / Collapsed / Hidden / HitTestInvisible / SelfHitTestInvisible, case-insensitive) that takes precedence over the `visible` bool, reaching all five Slate states. Unknown strings are rejected with `INVALID_VISIBILITY` instead of being silently coerced to false. The response now echoes the resolved `visibility` enum string alongside the `visible` bool. The handler reuses the existing `WidgetAuthoringHelpers::TryParseVisibility`/`VisibilityToString` (no reinvented enum table); the registered summary and a new `### ui.set_widget_visibility` overlay section now describe the actual param shape. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/UI/UiHandler.cpp` (handler body + summary + include of `WidgetAuthoringUtils.h`), `docs/wiki-src/ui.md` (new overlay section). Regression tests in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestUIHandlers.cpp`: `ui.set_widget_visibility.EnumParamRegistered` (asserts the `visibility` string param + back-compat `visible` bool are in the registered contract and the summary names Hidden/HitTestInvisible) and `ui.set_widget_visibility.EnumRoundTrip` (drives the production `TryParseVisibility`/`VisibilityToString` for all five states incl. the three the old bool couldn't reach, case-insensitive parse, and strict reject-on-unknown) — both fail if the fix is reverted.
- `#1-initial-audit` `OPEN` reporter — Live-HUD PIE preview task: `ui.set_widget_visibility` accepts only a plain bool `visible` and its response echoes only `visible:true|false`, while the method doc advertises the full `ESlateVisibility` enum. Passing `visible:'Collapsed'` was silently coerced to a bool (3 calls, all ok, none honoring the requested enum); `Hidden`, `HitTestInvisible`, and `SelfHitTestInvisible` are unreachable, and `Collapsed` vs `Hidden` (a real layout difference the task explicitly requested) cannot be expressed. Propose accepting the ESlateVisibility string, echoing the resolved enum state, rejecting unknown values, and aligning the `docs/wiki-src/ui.md` overlay with the actual schema.
