---
id: F-editor-undo-history
title: "No way to read the editor undo history, and editor.undo/editor.redo apply one step and report no title"
status: IN-REVIEW
severity: Medium
category: feature
tags: [editor, undo, redo, transactions, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-28T12:00:00Z
---

# No way to read the editor undo history, and undo/redo report no title

`editor.undo` and `editor.redo` (`Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp`)
call `GEditor->UndoTransaction()` / `RedoTransaction()` once and return only `{action, success}`.
An agent cannot see what it is about to undo, cannot tell which transaction a call reversed (its
own, or one another client or the user recorded in the shared editor), and must loop one call per
step. Nothing exposes the transaction buffer, although `UTransactor` publishes it read-only on
every supported engine (`GetQueueLength`, `GetUndoCount`, `GetTransaction(i)->GetContext()`,
`CanUndo`/`CanRedo` with a reason). Nothing to undo was reported as a success response carrying
`success:false`.

**Fix:** new read-only `editor.undo_history` (`limit`): `queueLength`, `undoCount`,
`transactionActive`, `canUndo`/`canRedo` with the engine's reason, `undo[]` newest first and
`redo[]` in redo order, each entry `{index, title, context, id, primaryObject?}`, plus
`undoTruncated`/`redoTruncated`. `editor.undo`/`editor.redo` gain `steps` (default 1) and return
`undone[]`/`redone[]` for the transactions actually applied, `requestedSteps`, `completedSteps`,
`stoppedReason`; zero steps applied is `NOTHING_TO_UNDO`/`NOTHING_TO_REDO` with that payload.

Engines: API identical on 5.3 to 5.8 (`Editor/Transactor.h`, `Editor/TransBuffer.h`).

**Related:** `B-no-undo-redo` (undo verbs switched to `UndoTransaction`).

## History
- `#1-gap-undo-history` `OPEN` reporter - Found in the 2026-09-28 competitive gap analysis: no undo-history read and no titles or step count on undo/redo. `UTransactor` read API verified unchanged across `C:\UE_5.3` to `C:\UE_5.8`. Severity Medium: a soft blocker (undo is blind to what it reverses in a shared editor; stepping needs a caller loop).
- `#2-undo-history-and-steps` `IN-REVIEW` developer - Added `editor.undo_history` and `steps` + `undone[]`/`redone[]` on `editor.undo`/`editor.redo` in `Handlers/Editor/EditorCommandHandler.cpp` (shared `EditorUndoHistory::RunSteps`; the applied entry is read at the position the buffer ended on, since one engine undo can skip expired transactions). New `ERR_NOTHING_TO_REDO`, reused `ERR_NOTHING_TO_UNDO` in `Handlers/ErrorCodes.h`; nothing-to-undo/redo is now that error instead of a success carrying `success:false`. Tests in `Tests/EditorOps/TestEditorUndoHistory.cpp` swap `GEditor->Trans` for a fresh `UTransBuffer` and restore it, so the editor's real history is untouched: `PinWright.editor.undo_history.ListsTitledTransactionsNewestFirst`, `PinWright.editor.undo_history.EmptyBufferIsATypedAnswer`, `PinWright.editor.undo.StepsUndoAndRedoReportTitles`, `PinWright.editor.undo.EmptyBufferIsNothingToUndo`. Wiki: `docs/wiki-src/editor.md` sections for the three verbs. Both files compile-checked with `-SingleFile` on 5.8; tests not run yet.
