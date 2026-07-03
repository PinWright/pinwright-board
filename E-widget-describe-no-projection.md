---
id: E-widget-describe-no-projection
title: "widget.describe has no field projection — a single-field read (root_class / child existence) returns the whole tree + binding-target monolith and spills to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [widget, widget-describe, response-size, projection, spill]
---

# widget.describe has no field/scope projection — single-field reads spill the whole tree

`widget.describe` always serializes the full widget tree (every node's `props`,
`slot`, `bindings`, `delegates`) plus a top-level `bindingTargets` array
harvested from *all* the blueprint's graphs, and emits `root_class` alongside
(`Source/PinWright/Private/Handlers/UI/WidgetDescribeHandler.cpp:546-563`). The
registered params only *add* to or *scope within* the tree — `widgetName` picks
a subtree, `max_depth` caps depth, `include_slot` / `include_bindings` /
`include_metadata` / `include_defaults` toggle per-node detail (`:163-203`,
`:273-278`) — none can drop the tree itself, and `tree` is built unconditionally
(`:552`). There is **no field projection** (`include:["meta"|"tree"|"bindings"]`,
a `metaOnly`/`treeOnly` flag, or a names-only/existence query) to return just the
part you need.

So a single-fact lookup — the BP's parent class, or whether one named child
exists — pays for the whole tree. On a real menu widget that overflows the
10000-char inline budget (`E-http-response-spill`, DONE), the payload spills to
`Saved/EditorAutomation/HttpResponses/...` and forces a `Read` + grep to extract
one field. Distinct from the existing `widget.describe` tickets, which are about
slot-value truncation (`E-widget-describe-slot-truncation`, DONE), the
TREE_EMPTY C++ hint (`E-widget-describe-cpp-hint`, DONE), and live-root
disambiguation (`F-widget-describe-live-root-disambiguation`, IN-REVIEW) — none
addresses response size / projection. Same shape as
`E-actor-describe-no-header-only-read` and `E-get-nodes-pins-spill-no-projection`
but a different method (the board files one projection ticket per verb).

**Session evidence (replayable):**
`call(method="widget.describe", args={widgetPath:"/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone"})`
→ `{"outputTooLong":true,"message":"Response exceeds display limit (102852 chars,
threshold 10000); full payload written to ...HttpResponses/...json"}`. The
102852-char payload had to be written to disk and grepped just to read
`root_class` and confirm two child widgets existed.

**Workaround:** let it spill, then `Read`/grep the HttpResponses JSON for the one
field (or pass `widgetName=<child>` to probe one subtree at a time — still
returns that subtree's full props and can't yield `root_class` alone).
**Fix:** add an optional, additive field projection to `widget.describe` — an
`include`/`fields` allow-list over the top-level keys
(`root_class`/`tree`/`bindingTargets`/`widget_count`) plus a `namesOnly` tree
mode emitting just `name`/`type`/`children` — mirroring the `fields`/`namesOnly`
levers proposed in `E-actor-describe-no-header-only-read` and
`E-get-nodes-pins-spill-no-projection`. Default output stays byte-identical.

## History
- `#1-initial-repro` `OPEN` reporter — Verified at source: `WidgetDescribeHandler.cpp:546-563` unconditionally assembles `root_class` + full recursive `tree` (per-node props/slot/bindings/delegates) + all-graphs `bindingTargets` + `widget_count`; the only levers (`:163-203`, `:273-278`) are `widgetName` subtree scope, `max_depth`, and `include_slot`/`include_bindings`/`include_metadata`/`include_defaults` per-node toggles — all additive/scoping, none drops the tree or yields meta-only. No `fields`/`include`/`metaOnly`/`treeOnly`/`namesOnly` projection exists. Replayable: `widget.describe {widgetPath:"/App/App/UI/LobbyAndMenu/W_DroneSelect_EditDrone"}` → `outputTooLong` at 102852 chars (threshold 10000), spilled to `HttpResponses/...json`, then grepped just to read `root_class` and confirm two child widgets existed. Severity Low per rubric (a response spill that only forces a Read); no reach bump (widget.describe runs in widget-authoring sessions, not almost every session). Dedup: the three existing widget.describe tickets (slot-truncation DONE, cpp-hint DONE, live-root-disambiguation IN-REVIEW) cover unrelated aspects; `E-actor-describe-no-header-only-read` / `E-get-nodes-pins-spill-no-projection` are the same projection shape on different verbs and the board files one such ticket per method.
