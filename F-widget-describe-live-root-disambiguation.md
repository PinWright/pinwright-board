---
id: F-widget-describe-live-root-disambiguation
title: "widget.describe capture_source=live cannot disambiguate multiple live UMG roots — AMBIGUOUS_LIVE_ROOT with no working selector"
status: IN-REVIEW
severity: Medium
category: feature
tags: [widget, describe, live, pie, ambiguous-live-root, disambiguation]
---

# widget.describe capture_source=live cannot disambiguate multiple live UMG roots

`B-widget-describe-live-no-root` (DONE) fixed the single-root case — live
capture now formats the tree instead of collapsing to `Widget: Unknown`. But
its body explicitly flagged the multi-root follow-up as still open:

> For multi-rooted UI (game viewport layer + menu layer + modal layer in
> Lyra), either return all live UMG roots in an array, or accept an optional
> `instance_name` / `root_index` to pick one.

That selector capability was never implemented. When **two or more** live UMG
roots are on screen — common as soon as a level's own HUD plus a
`ui.create_hud`-instantiated HUD coexist in the PIE viewport —
`widget.describe capture_source=live` aborts with `AMBIGUOUS_LIVE_ROOT` and
offers no working way to pick one, so the caller cannot read back live state to
verify the values it just set. This blocks the "confirm the final on-screen
state" close-out that live-HUD tasks ask for.

## Repro (live, from a HUD PIE preview task)

1. `editor.play` (the level auto-adds its own `WBP_PlayerHUD_C_0`).
2. `ui.create_hud WBP_PlayerHUD_StateTree` → painted as `..._StateTree_C_0`.
3. `widget.describe { capture_source: "live" }` (targeting the StateTree
   instance) →
   `[AMBIGUOUS_LIVE_ROOT] Multiple live UMG root candidates matched:`
   `SConstraintCanvas -> SObjectWidget (WBP_PlayerHUD_C_0),`
   `SConstraintCanvas -> SObjectWidget (WBP_PlayerHUD_StateTree_C_0)`
4. Disambiguation attempt — top-level `instance_name` →
   `[UNKNOWN_PARAMS] Unknown parameter(s) for 'widget.describe': [instance_name].`
   (`instance_name` is not among the ~20 accepted params.)
5. Disambiguation attempt — `resolve_geometry.instance_name` (the precedent the
   DONE ticket pointed at) → same `AMBIGUOUS_LIVE_ROOT`; the nested
   `instance_name` selects geometry resolution, not which live root to dump.

So neither selector path the DONE ticket suggested actually disambiguates the
root: one is rejected as unknown, the other is ignored for root selection.

## What it should do

Provide a way to resolve the ambiguity, e.g. one of:
- Accept a top-level `instance_name` (or `root_index`) on `widget.describe`
  that selects which live UMG root to dump (the same name `ui.create_hud`
  returns and `ui.remove_widget_from_viewport`'s `key` accepts —
  `WBP_PlayerHUD_StateTree_C_0` here), OR
- When ambiguous and no selector is given, return **all** live roots as an
  array (the other option the DONE ticket offered) so the caller can pick the
  subtree by name themselves.

The error already enumerates the candidate names; the caller just needs a
parameter that consumes one of them. Echo the resolved root in the response.

## Evidence

Attempt agent's friction note, verbatim:

> widget.describe capture_source=live can't disambiguate between two live
> roots: AMBIGUOUS_LIVE_ROOT fired and neither a top-level instance_name
> (UNKNOWN_PARAMS) nor resolve_geometry.instance_name worked, so I couldn't
> read back live state to verify values.

Three failed `widget.describe` live calls in the log (one AMBIGUOUS_LIVE_ROOT,
one UNKNOWN_PARAMS for `instance_name`, one AMBIGUOUS_LIVE_ROOT via
`resolve_geometry.instance_name`). Follow-up to DONE
`B-widget-describe-live-no-root` (#38 anticipated this exact multi-root case).

**Workaround:** none observed — with two live roots, there is no way to read
back the specific instance's live state through `widget.describe`.

## History
- `#1-initial-audit` `OPEN` reporter — Live-HUD PIE preview task with two live UMG roots (level's `WBP_PlayerHUD_C_0` + create_hud's `WBP_PlayerHUD_StateTree_C_0`). `widget.describe capture_source=live` returned `AMBIGUOUS_LIVE_ROOT` listing both candidates; top-level `instance_name` was rejected `UNKNOWN_PARAMS`, and `resolve_geometry.instance_name` still hit `AMBIGUOUS_LIVE_ROOT` (it scopes geometry, not root selection). 3 failed live describe calls; the caller could not read back live state to verify the values it had just set. The DONE `B-widget-describe-live-no-root` body proposed exactly this selector/array capability but only the text-formatter (single-root) part shipped. Propose a top-level `instance_name`/`root_index` selector (consuming the names the error already lists, matching `ui.create_hud`'s returned key) or returning all roots as an array when ambiguous.
- `#3-multi-root-selector-implemented` `IN-REVIEW` developer — Added a top-level `instance_name`/`root_index` selector to `widget.describe` and `widget.export_xml` for `capture_source=live`, resolving the `AMBIGUOUS_LIVE_ROOT` dead-end. Factored the root-pick logic out of `FLiveUiSnapshotService::Capture` into a new pure, testable `FLiveUiSnapshotService::SelectRootCandidate(CandidateNames, Request, ...)`: with no selector the single-root case still resolves and multi-root still returns `AMBIGUOUS_LIVE_ROOT` (now with a message naming the `instance_name`/`root_index` params to pass); `instance_name` does a substring match against the candidate backing-widget names (mirroring the `resolve_geometry.instance_name` `GetName().Contains` precedent, so the key `ui.create_hud` returns works), reporting `LIVE_ROOT_NOT_FOUND` on no match and `AMBIGUOUS_LIVE_ROOT` if the name still matches >1; `root_index` selects positionally (0-based), `LIVE_ROOT_NOT_FOUND` when out of range. `Capture` echoes the resolved root via new snapshot fields surfaced in the JSON writer as `selected_root_name`/`selected_root_index`/`root_candidate_count`. New request fields `InstanceName`/`RootIndex` on `FLiveUiSnapshotRequest`. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/UI/LiveUiSnapshot.h` (+request/snapshot fields, `SelectRootCandidate` decl), `.../LiveUiSnapshot.cpp` (`SelectRootCandidate` impl, `ResolveCandidateName` helper, `Capture` rewire), `.../LiveUiSnapshotJsonWriter.cpp` (echo fields), `.../WidgetDescribeHandler.cpp` and `.../WidgetXmlExportHandler.cpp` (param specs + request population). Test: `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestLiveUiSnapshotRootSelection.cpp` exercises `SelectRootCandidate` directly with the ticket's two-root repro names (single-root, multi-root-no-selector→AMBIGUOUS, instance_name resolves, full create_hud key, no-match→NOT_FOUND, over-broad name→AMBIGUOUS, root_index resolves, out-of-range→NOT_FOUND, empty→LIVE_UI_NOT_FOUND) — deterministic, no live PIE needed; it fails if the pre-fix unconditional ambiguity abort is restored. Not compiled here (later phase).
- `#2-aggregation-inspect-deadend-and-destructive-workaround` `OPEN` reporter — Independent live replay (`ui.set_widget_text` HUD prototype task: authored `/Game/UI/W_StatusHUD` with `StatusLabel`/`ScoreLabel`, instantiated via `ui.create_hud` → `W_StatusHUD_C_0`, alongside the level's pre-existing `WBP_PlayerHUD_C_0`). Same `AMBIGUOUS_LIVE_ROOT` on `widget.describe capture_source=live` listing both `SObjectWidget` candidates. Adds two distinct PROCESS data points the #1 note lacked: (a) the **discoverability dead-end** the error pushes callers into — after the ambiguity fired, the agent tried `system.inspect.find_by_class` twice (short name `W_StatusHUD_C` and full path `/Game/UI/W_StatusHUD.W_StatusHUD_C`, both → 0) and read the `system.inspect.list_objects` / `runtime-uobject-inspection` wiki, all of which are **actor-only** and cannot locate a live `UUserWidget`, so there is no listing primitive that surfaces the candidate names outside the error string itself; (b) a **destructive workaround does exist** (contra #1's "none observed") — the agent called `ui.remove_widget_from_viewport key=WBP_PlayerHUD_C_0` to evict the OTHER live root, leaving its own HUD as the sole root, after which live describe succeeded (`StatusLabel.Text=PLAYING`, `ScoreLabel Collapsed`). That workaround mutates the scene under test (removes a widget the caller may need) and is unavailable when both roots must stay on screen. Reinforces the #1 proposal: a top-level `instance_name`/`root_index` selector (or all-roots array) would avoid both the inspect dead-end and the destructive evict. Consider also having the `AMBIGUOUS_LIVE_ROOT` message name the parameter to pass once it exists.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
