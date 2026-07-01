---
id: E-widget-describe-slot-truncation
title: "`widget.describe` truncates Slot values with an ellipsis, hiding layout data"
status: DONE
severity: Low
category: ergonomic
tags: [widget-describe, slot, truncation]
---

# `widget.describe` truncates Slot values with an ellipsis, hiding layout data

`widget.describe` with `include_slot=true` claims to return slot/layout properties, but the serialised slot string is truncated mid-path with `...` so actual coordinate values are never visible.

**Observed output** (this session, describe on `W_AppReplayEditor` with `include_slot: true`):

```
CanvasPanel "RootCanvas"
  W_ReplayEditor_ClipSettingsPanel_C "ClipPanel" {Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}
  W_ReplayEditor_Toolbar_C "Toolbar" {Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}
  W_ReplayEditor_Transport_C "Transport" {Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}
  W_ReplayEditor_Timeline_C "Timeline" {Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}
```

Only the internal package path is printed, then cut off. No anchors, offsets, or alignment values reach the caller. The same widget exported via `widget.export_xml` shows the full layout cleanly:

```
Slot.LayoutData="(Offsets=(Right=400.000000,Bottom=240.000000),Anchors=(Minimum=(X=0.000000,Y=0.000000),Maximum=(X=0.000000,Y=1.000000)),...)"
```

**Impact:** Anyone verifying canvas layout via `widget.describe` (the natural first-reach inspection tool) gets nothing and has to fall back to `widget.export_xml`. That makes the `include_slot` param near-useless for layout verification.

**Workaround:** Use `widget.export_xml` instead.

**Proposal:** In the describe output path, print parsed slot layout fields (Anchors, Offsets, Alignment, Size, ZOrder, etc.) as structured key=value pairs instead of the raw internal package path. The XML exporter already has this formatting logic; reuse it. If the full raw string is needed for debugging, put it behind a separate `include_raw_slot` flag.

## History
- `#1-slot-truncation-repro` `OPEN` reporter — Verified root replay-editor layout after composing four sub-widgets via `widget.set` with `slot` JSON. `widget.describe include_slot=true` returned the truncated `{Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}` for every child; had to rerun with `widget.export_xml` to confirm the anchors/offsets were applied correctly. The describe call contributed zero information on what it was specifically asked to include.
- `#2-investigated-no-repro` `OPEN` developer — Investigated at source during mcp-sprint. Handler at `WidgetDescribeHandler.cpp:385-395` serialises slot as structured JSON: `slot: { type: string, props: { <FProperty>: <JsonValue> } }` via `CollectOverriddenProperties`. `ExportPropertyToJsonValue` in `PropertyUtils.cpp:122-129` emits object references as full-path strings (no truncation). Grep for `{Slot=` across plugin source: zero hits. The reporter's `{Slot=/App/App/UI/ReplayEditor/W_AppReplayE...}` textual format is not produced by this plugin and is almost certainly the MCP client's rendering of a long object-reference string. Attempted live verification via `mcp__editor_automation__.call path="widget.describe" args={...}` during sprint but the editor/gateway was unreachable (HTTP 000). Staying OPEN pending live repro — either the user reruns `widget.describe` in a session with a live editor to confirm structured JSON (→ WONTFIX), or surfaces a counterexample (→ reshape with real repro). No code change attempted.
- `#3-structured-slot-text-format` `IN-REVIEW` developer — Changed widget.describe text formatting to print structured slot data from node.slot instead of truncating the raw Slot object path, and added FWidgetDescribeTextFormatterShowsStructuredSlotTest to guard layout fields in text output.
- `#4-verified-slot-text` `DONE` tester — Verified: `widget.describe` on `/Game/App/UI/Test/W_McpReviewTemp_20260429` subtree `PanelA` with `include_slot:true`, `max_depth:2` printed child slot lines such as `[slot] HorizontalBoxSlot {}` for `AlphaText`, `GammaText`, `BetaText`, and `NestedHotkeyRow` without the old object-path ellipsis truncation.
