---
id: B-game-view-reports-invariant-show-flags
title: "set_game_view reports overlay show-flags that are structurally constant — billboardSprites can never read false, and reading it as 'billboards still on' has already caused an already-fixed High defect to be re-reported"
status: OPEN
severity: Medium
category: bug
tags: [editor, set_game_view, show-flags, billboard, response-semantics, invariant-field, false-signal, docs]
encounters: 1
costly: 1
lastSeen: 2026-08-20T00:00:00Z
---

# A field that cannot vary, next to a summary that promises it will

The verb's summary (`Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp:614`) says game
view "hides editor-only sprites/icons/billboards so the viewport renders like the shipped game", and
the same response carries `overlayShowFlags.billboardSprites`, read live off the flag at `:599`
(`Out->SetBoolField(TEXT("billboardSprites"), Flags.BillboardSprites != 0);`, emitted as
`overlayShowFlags` at `:687` and `previous.overlayShowFlags` at `:665`/`:681`). After a confirmed
enable it reads `true`.

That is not a behaviour contradiction, and the original report's framing needs correcting.
`BillboardSprites` is declared `SHOWFLAG_FIXED_IN_SHIPPING(1, BillboardSprites, ...)` —
`C:\UE_5.8\Engine\Source\Runtime\Engine\Public\ShowFlagsValues.inl:213` — and the `FEngineShowFlags`
constructor memsets **every** flag on before selectively disabling a list that never includes it
(`ShowFlags.h:389-397`, "Most flags are on by default. With the following line we only need disable
flags"). `SetBillboardSprites` is never called for any `EShowFlagInitMode`, so
`FEngineShowFlags(ESFIM_Game).BillboardSprites == 1` exactly as in `ESFIM_Editor`.

Editor billboards are suppressed in game view by the **`Editor`** flag instead — `ShowFlags.h:463`,
`SetEditor(InitMode == ESFIM_Editor || InitMode == ESFIM_VREditing)` — which this response never
reports.

## The cost is real and already paid once

The field cannot read `false` in any game flag set, so it carries no information, and it invites the
reader to conclude the toggle failed. That is the same conclusion that produced
`B-set-game-view-keeps-editor-billboards` (IN-REVIEW, High) — whose actual defect, a phantom
`GEditor->Exec("ToggleGameView 1")` no-op with an unconditional `gameViewEnabled: true`, is **fixed
in this tree**: `:667` calls `ViewportClient->SetGameView(bEnabled)`, `:671` forces a redraw, and
`:676-677` reads back the real `IsInGameView()`, guarded by
`Tests/EditorOps/SetGameViewTogglesTest.cpp`. This ticket exists because the response field has now
sent an investigation back toward that closed defect.

## Three more invariants in the same block

Of the ten reported flags, only six vary with init mode.

| field | why it is constant |
|---|---|
| `billboardSprites` | never disabled for any init mode (above) |
| `selectionOutline` | `SetSelectionOutline(false)` unconditionally |
| `navigation` | `SetNavigation(false)` unconditionally |
| `modeWidgets` | forced true by `SetGameView` itself on enable — `EditorViewportClient.cpp:7259` |

Varying: `splines`, `selection`, `grid`, `volumes`, `lightRadius`, `audioRadius`.

## The one flag that is handled well

`:708-720` emits `overlayWarning` when `IsInGameView()` is true and `Splines` is still set, citing
`SetGameView`'s reuse of the current flags as the game set — confirmed at
`EditorViewportClient.cpp:7229-7240` (`if (EngineShowFlags.Game) { GameFlags = EngineShowFlags; }`).
That is the right shape. Nothing equivalent tells a reader that `billboardSprites` is meaningless,
and nothing should have to: the field should not be reported as if it were evidence.

**Fix:** drop the four invariant fields, or annotate them as invariant in the response, and report
`EngineShowFlags.Editor` — the flag that actually governs the sprites the summary promises to hide.
Reporting the governing flag makes the summary checkable instead of merely plausible.

## Related

- `B-set-game-view-keeps-editor-billboards` (IN-REVIEW, High) — the original defect, fixed in-tree at
  `:667`/`:676-677`. This ticket is about the response field that keeps pointing back at it.
- `B-set-game-view-does-not-suppress-spline-overlays` (OPEN, Medium) — the sibling: `99eedf35`
  shipped the `notGovernedByGameView[]` honesty, suppression still open. Guarded by
  `Tests/EditorOps/TestSetGameViewOverlayReporting.cpp`.

## History
- `#1-invariant-flag-reads-as-evidence` `OPEN` reporter — `editor.set_game_view`'s summary (`ViewportHandler.cpp:614`) states game view "hides editor-only sprites/icons/billboards", and the same response carries `overlayShowFlags.billboardSprites` read live off `EngineShowFlags.BillboardSprites` (`:599`, emitted `:687`), which reads `true` after a confirmed enable. Correction to the report: this is not a behavioural contradiction. `BillboardSprites` is declared with default 1 (`ShowFlagsValues.inl:213`) and the `FEngineShowFlags` constructor turns every flag on before selectively disabling a list that never includes it (`ShowFlags.h:389-397`), so the flag is 1 in `ESFIM_Game` exactly as in `ESFIM_Editor`; editor billboards are suppressed by the `Editor` flag instead (`ShowFlags.h:463`), which the response never reports. The field therefore cannot read `false` in any game flag set, carries no information, and invites the reader to conclude the toggle failed — the same conclusion that produced `B-set-game-view-keeps-editor-billboards` (IN-REVIEW, High), whose real defect (a phantom `GEditor->Exec("ToggleGameView 1")` plus an unconditional `gameViewEnabled:true`) is fixed in this tree at `:667` with an honest `IsInGameView()` readback at `:676-677` and `Tests/EditorOps/SetGameViewTogglesTest.cpp` guarding it. Three more of the ten reported flags are structurally constant for the same reason — `selectionOutline` and `navigation` are set false unconditionally, and `modeWidgets` is forced true by `SetGameView` itself (`EditorViewportClient.cpp:7259`) — leaving only `splines`, `selection`, `grid`, `volumes`, `lightRadius`, `audioRadius` varying. The `overlayWarning` at `:708-720` for `Splines` is the right shape and is correctly grounded in `EditorViewportClient.cpp:7229-7240`. Fix: drop or annotate the four invariant fields and report `EngineShowFlags.Editor`, the flag that actually governs the sprites the summary promises to hide. Severity Medium rather than Low because the field has already cost one re-investigation of a closed High defect.
