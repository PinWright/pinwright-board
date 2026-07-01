---
id: E-activatable-pop-transition-readback-timing
title: "`ui.activatable_pop` / `list_stack_widgets` / `get_active_widget` — default CommonUI transition (~0.4s) defers post-pop removal; immediate readback can be stale, undocumented"
status: WONTFIX
severity: Medium
category: ergonomic
tags: [ui, activatable_pop, list_stack_widgets, get_active_widget, common-ui, docs, readback]
encounters: 1
lastSeen: 2026-06-30T22:53:48.7445084+03:00
---

# CommonUI's default transition defers post-pop removal — an immediate `ui.activatable_*` readback can lie

`UCommonActivatableWidgetContainerBase` runs a transition animation
(`TransitionDuration`, **default ~0.4s**) on push/pop. On a pop, the popped widget
is not removed from the stack until that transition completes — so a
`ui.list_stack_widgets` / `ui.get_active_widget` readback issued **immediately
after** `ui.activatable_pop` can still report the just-popped widget as present /
active. The caller, seeing the popped entry still on top, can reasonably conclude
the pop failed and pop again (or fail their verification). None of the
`ui.activatable_pop` / `ui.list_stack_widgets` / `ui.get_active_widget` wiki pages
mention this transition-vs-readback timing.

The agent had to leave the wiki for **engine source**
(`CommonActivatableWidgetContainer.h`) to learn the behavior, then proactively
authored the host stack with `TransitionDuration=0` (via `widget.set`) so its
post-pop `list_stack_widgets` / `get_active_widget` readbacks were unambiguous.
That source-dive + defensive mitigation is process friction even though the task
came out clean — an agent who didn't know to zero the transition (i.e. used the
stock default) would get the misleading readback.

## Evidence (friction note, verbatim)

> "One wiki gap: I set TransitionDuration=0 proactively (learned from the engine
> CommonActivatableWidgetContainer.h, since the activatable-stack push/pop wiki
> pages never mention that the default 0.4s transition defers post-pop removal and
> could make an immediate list/get_active readback ambiguous)."

The same engine header dive also served the `widget.add` type-string discovery
(filed separately as `E-widget-add-type-class-discovery`) — one source-dive
covering two undocumented gaps.

## What it should do (downstream, wiki only)

Improve `docs/wiki-src/ui.md`. The `### ui.activatable_pop` /
`### ui.list_stack_widgets` / `### ui.get_active_widget` H3 sections (added by
`E-activatable-host-runtime-instance-name-undocumented`, IN-REVIEW) should carry a
timing note: the host container's `TransitionDuration` (default ~0.4s) defers
post-pop removal, so a `list_stack_widgets` / `get_active_widget` readback issued
in the same tick as `activatable_pop` may still reflect the pre-pop stack. Tell
authors to either set the stack's `TransitionDuration=0` for deterministic
readback (one `widget.set` at author time), or poll the readback until the count /
active widget settles. Optionally `ui.activatable_pop` could return a
`transitionPending` / `removalDeferred` hint when the host's transition is non-zero
so the caller knows a readback may lag.

severity rationale: impact=misleading/stale readback the caller can trust (pop
appears to have failed) — avoided here only by an engine-source dive to learn the
mitigation (a blocker-with-workaround) × reach=rare (CommonUI activatable pop +
same-tick readback) -> Medium.

## History
- `#2-wontfix` `WONTFIX` developer — Not worth the docs churn. The engine behavior is real (`UCommonActivatableWidgetContainerBase` defers post-pop removal until the ~0.4s `TransitionDuration` transition finishes; `DeactivateWidget()` then `ReleaseWidget`/`WidgetList.Remove` post-transition), but the reported friction is practically unreachable across the MCP transport: `ui.activatable_pop`, `ui.list_stack_widgets`, and `ui.get_active_widget` are DISTINCT RPCs. A client waits for pop's response, does model inference, then issues the readback — seconds of wall-clock during which PIE keeps ticking and the sub-0.4s transition completes. The stale window (<0.4s) is far shorter than any realistic inter-RPC gap, so a same-response-cycle stale readback does not arise in the agent-ergonomics scenario this ticket targets. It was never observed (the run was MCP-clean, 19 RPCs, no retries) — the reporter pre-emptively zeroed the transition after an engine source-dive, and a trivial reporter-known workaround already exists (`SetTransitionDuration`/author-time `TransitionDuration=0`). A "readback may lag 0.4s" note risks the opposite harm: nudging agents to add unnecessary polling/waits for a condition that does not manifest across separate RPCs. The pop handler's own success payload is never stale — it captures `RemovedName` before `RemoveWidget` (UiActivatableStackHandler.cpp:190-191). WONTFIX (adversarial lens).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit PROCESS finding from a CommonUI modal-layer-host push/pop/readback task (`focus: ui.activatable_push`). The run was MCP-clean (19 RPCs, no retries), but the friction note records the agent source-diving `CommonActivatableWidgetContainer.h` to learn that CommonUI's default ~0.4s `TransitionDuration` defers post-pop removal, then authoring the host stack with `TransitionDuration=0` so the post-pop `list_stack_widgets` (1 entry, CenterPopup active) / `get_active_widget` (CenterPopup) readbacks were unambiguous. The four `ui.activatable_*` wiki sections never warn about this transition-vs-readback timing, so an agent using the stock default could read a stale top after pop and conclude the pop failed. Distinct from the two existing activatable docs tickets on the same `ui.md` page: `E-activatable-push-requires-c-suffix` (`widgetClass` `_C` suffix) and `E-activatable-host-runtime-instance-name-undocumented` (`host` = live instance name) — both about *which string to pass*, neither about *when the readback is valid*. Fix is a `docs/wiki-src/ui.md` overlay (timing note on the pop/list/get-active H3 sections) only.
