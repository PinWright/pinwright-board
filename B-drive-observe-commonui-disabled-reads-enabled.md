---
id: B-drive-observe-commonui-disabled-reads-enabled
title: "drive.observe reports enabled:true / interactable:true for a disabled UCommonButtonBase, and drive.expect widget_enabled is met on it — Element.bEnabled reads SWidget::IsEnabled(), which CommonUI's disable path never writes"
status: OPEN
severity: Medium
category: bug
tags: [drive, drive.observe, drive.expect, widget_enabled, commonui, ucommonbuttonbase, enabled-state, false-positive, pie]
encounters: 1
costly: 1
lastSeen: 2026-09-09T14:00:00Z
---

# A CommonUI button disabled with `SetIsEnabled(false)` is reported as enabled and interactable

UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`. Real-click PIE run on the
`L_Core` main menu -> `W_Multiplayer` -> `W_CreateMultiplayerRoom` popup.

A `UCommonButtonBase` that had been disabled via `UCommonButtonBase::SetIsEnabled(false)`
was visually greyed and functionally inert (a real, OS-level mouse click on it produced no
state change and no log line at all). Yet:

- `drive.observe` listed it with `enabled:true, interactable:true`.
- `drive.expect {condition:{type:"widget_enabled", target:"Create Small Sumo"}}` returned
  `met:true`.

So both the observation and the assertion verb affirm a button that cannot be pressed. The
only way the disabled state could be established was **out of band**: reading the greying
from the screenshot pixels, plus the absence of the launch log line after clicking.

## Why it happens (verified in source, both sides)

The drive element snapshot takes the enabled flag straight off the Slate widget:

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveLiveResolver.cpp:339`
  — `Element.bEnabled = SlateWidget->IsEnabled();` (game/UMG surface)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveEditorChrome.cpp:147`
  — the same line for the editor-chrome surface.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveConditionEval.cpp:166-180`
  — `widget_enabled` is just `Result.bMet = Match->bEnabled;`, i.e. the identical snapshot,
  which is why the expectation inherits the same wrong value.

`SWidget::IsEnabled()` returns that **one widget's own** enabled attribute. Slate never writes
a parent's disabled state down into children — propagation happens at paint/hit-test time
through the `bParentEnabled` argument (`SWidget::ShouldBeEnabled(bool InParentEnabled)`,
`C:\UE_5.8\Engine\Source\Runtime\SlateCore\Public\Widgets\SWidget.h:1724`). So a leaf under a
disabled ancestor keeps reporting `IsEnabled() == true`.

CommonUI then makes this worse by keeping its own disable state entirely off the Slate enabled
attribute (engine source under
`C:\UE_5.8\Engine\Plugins\Runtime\CommonUI\Source\CommonUI\Private\`):

- `CommonButtonBase.cpp:563` `UCommonButtonBase::SetIsEnabled` — sets `bButtonEnabled`, calls
  `Super::SetIsEnabled` with broadcasting suppressed (its own comment: "do not broadcast
  because we don't want to propagate it to the underlying SWidget"), then `DisableButton()`.
- `CommonButtonBase.cpp:2345-2353` `DisableButton()` -> `RootButton->SetButtonEnabled(false)`
  -> `CommonButtonBase.cpp:194-201` `UCommonButtonInternalBase::SetButtonEnabled` ->
  `CommonButtonTypes.cpp:162-165` `SCommonButton::SetIsButtonEnabled`, which assigns the
  private `bIsButtonEnabled` and **nothing else** — there is no `SWidget::SetEnabled` call
  anywhere on that path.
- The authoritative predicates are therefore invisible to `IsEnabled()`:
  `CommonButtonTypes.cpp:194` `SCommonButton::IsInteractable() = bIsButtonEnabled && bIsInteractionEnabled`
  (and `Press()` at `:155-159` gates on it), `CommonButtonTypes.cpp:198-201` `OnPaint` greys
  with `bEnabled = bParentEnabled && bIsButtonEnabled` (that greying is what the screenshot
  showed), and on the UMG side `CommonButtonBase.cpp:863-866`
  `UCommonButtonBase::IsInteractionEnabled() = GetIsEnabled() && bButtonEnabled && bInteractionEnabled && visible`.

Both mechanisms point the same way: the field drive reports is the one CommonUI deliberately
leaves alone.

## Impact

An agent driving a CommonUI menu cannot tell a live button from a dead one:

- It clicks a disabled button, gets `no_change_within_budget`, and — because observe swore the
  button was enabled and interactable — concludes the **input path** is broken. That is the
  exact shape of `B-drive-click-misses-pie-game-viewport`, filed earlier the same day, whose
  `#1` history used "a disabled quick-host button" as a control. Any diagnosis leaning on
  `enabled:true` from observe is standing on this defect.
- `widget_enabled` cannot be used as a gate ("wait until the Create button becomes clickable"),
  because it is met from the first frame.

## What it should do

For `UCommonButtonBase` and its subclasses, derive `enabled` from CommonUI's own predicate —
`IsInteractionEnabled()`, which already folds in `GetIsEnabled()`, `bButtonEnabled`,
`bInteractionEnabled` and visibility — rather than from the backing SWidget's attribute; the
pure-Slate equivalent is `SCommonButton::IsInteractable()`. `widget_enabled` must use the same
rule, since it reads the same field.

Worth doing at the same time, because it is the general form of the same bug: an element under
a disabled *ancestor* also reports `enabled:true` today. The resolver already walks the tree
top-down (`DriveLiveResolver.cpp:352-365` `WalkAndCollect`), so folding the walked ancestors'
enabled state into `Element.bEnabled` fixes the non-CommonUI cases too.

`interactable:true` is a separate, weaker claim — `FDriveLiveResolver::IsLikelyInteractable`
(`DriveLiveResolver.cpp:599-630`) is a pure type-name test ("is this the kind of widget one acts
on"), not a liveness test — so reporting it for a disabled button is arguably correct. It is
only misleading in combination with the wrong `enabled`; fixing `enabled` is enough.

**Workaround:** read the CommonUI state by reflection instead of trusting the drive snapshot —
call `IsInteractionEnabled()` on the `UCommonButtonBase` instance — or infer it from pixels (the
greying) plus the absence of the expected log line, which is what this session had to do.

severity rationale: impact = silent wrong data on a normal path that the caller acts on (the
rubric's High band: a `false` reported as `true` for the one flag deciding whether a click can
work, and it can manufacture a false "the click path is broken" conclusion) x reach =
`drive.observe`/`drive.expect` run in nearly every drive session, but only CommonUI-derived
buttons in a disabled state are misreported, a subset of surfaces -> bumped down one to Medium.
If a fixer confirms the same wrong value on the plain parent-disabled path (the generalization
above, which affects every widget kind), the reach bump-down no longer applies and this is High.

## History
- `#1-commonui-disabled-reads-enabled` `OPEN` reporter — Filed from a real-click PIE verification (UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`, `L_Core` -> `W_Multiplayer` -> `W_CreateMultiplayerRoom`, 2560x1440 at 150% Windows scaling). A `UCommonButtonBase` disabled through `SetIsEnabled(false)` was greyed and inert (a real click produced no effect and no log line), but `drive.observe` reported `enabled:true, interactable:true` and `drive.expect widget_enabled "Create Small Sumo"` returned `met:true`. Root cause read verbatim on both sides: drive snapshots `SWidget::IsEnabled()` (`DriveLiveResolver.cpp:339`, `DriveEditorChrome.cpp:147`) and `widget_enabled` reuses that same field (`DriveConditionEval.cpp:166-180`), while CommonUI's disable path never writes it — `UCommonButtonBase::SetIsEnabled` (`CommonButtonBase.cpp:563`) suppresses propagation to the SWidget by design and the chain ends at `SCommonButton::SetIsButtonEnabled` (`CommonButtonTypes.cpp:162-165`) assigning a private bool consulted only by `IsInteractable()` (`:194`) and `OnPaint` (`:198-201`). Marked `costly`: establishing the true state needed an out-of-band screenshot read plus a log check, and this is the class of wrong value that produces a false input-path diagnosis — `B-drive-click-misses-pie-game-viewport` `#1` used a disabled button as its control.
