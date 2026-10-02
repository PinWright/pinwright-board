---
id: B-material-auto-layout-misreports-and-no-undo
title: "material.authoring.auto_layout reports expressionsLaidOut = total expression count even when it moved nothing, and its moves are not undoable (no transaction, no Modify)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [layout, material, mgir, response-honesty, undo, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# material.authoring.auto_layout misreports its work and cannot be undone

The handler is at `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:4449-4495`.

1. **Misreports (rpc-design §1).**
   - It counts every expression *before* the layout (`:4466-4475`) and returns that
     number as `expressionsLaidOut` (`:4490`).
   - `FMGIRLayoutEngine` moves only expressions still at (0,0)
     (`MGIRLayoutEngine.cpp:13-18, 61-64`).
   - So on an already-positioned graph the verb moves nothing and reports
     `expressionsLaidOut: N`. A caller relying on that number believes the graph
     was re-flowed.
   - `F-material-graph-auto-layout`'s DONE verification (`expressionsLaidOut:6`)
     cannot tell the two cases apart.
2. **Not undoable.** There is no `FScopedTransaction` and no per-expression
   `Modify()`. `PreEditChange` / `PostEditChange` / `MarkPackageDirty` do not make
   the moves undoable.
3. **Open editor.** It passes `bCheckEditorOpen=false` (`:4460-4461`), but an open
   Material Editor works on its own copy of the material. Whether that editor later
   overwrites the moves is unverified.

**Fix:**
- Report `movedCount` and `moved[{nodeId, from, to}]` from positions read back
  after the move, plus the unchanged count.
- Wrap the call in a transaction, `Modify()` each moved expression, and cancel the
  transaction when nothing moved.
- Add the shared required `scope` parameter (`"unpositioned"` keeps today's
  behaviour), per the contract in `F-blueprint-auto-layout-rpc`.
- Either route the moves through the open editor's `UMaterialGraph`, or refuse
  with a named error.

**Acceptance:**
- On a fully positioned material the verb returns `movedCount: 0`.
- On a material with 2 unpositioned expressions it returns `movedCount: 2`, and
  their readback positions are non-zero.
- An editor undo restores the pre-call positions.
- A test covers the failure direction: a count that does not match the readback
  fails.

## Severity justification

**Medium.** By impact class this is a silent false-success, which is High. Reach
is low (the standalone verb is rarely called), so it drops one level.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30 (code reading): expressionsLaidOut is the pre-layout total expression count (MaterialAuthoringHandler.cpp:4466-4490) while the engine only moves (0,0) expressions; no transaction or Modify(), so moves cannot be undone; open-editor copy interaction unverified.
- `#2-readback-report-and-transaction` `IN-REVIEW` developer — `material.authoring.auto_layout` (Handlers/Material/MaterialAuthoringHandler.cpp) now snapshots every expression position before the layout and reports from the positions read back after it: `movedCount`, `unchangedCount`, `moved[{nodeId (expression GUID), from:{x,y}, to:{x,y}}]`; `expressionsLaidOut` is removed. The layout (Layout/PwGraphLayoutMaterial.cpp `ArrangeAndWrite`, the new PwGraphLayout core) already `Modify()`s each expression it moves and only moves (0,0) expressions; the handler now wraps it in one `FScopedTransaction` ("PinWright: material.authoring.auto_layout", echoed as `transaction`), cancels it and skips PreEditChange/PostEditChange/MarkPackageDirty when nothing moved. Tests (Tests/Material/TestMaterialAuthoringAutoLayout.cpp): `PinWright.material.authoring.auto_layout.ReflowsUnpositionedExpressions` (2 unpositioned -> movedCount 2 == readback count, moved[].to == live positions), `PinWright.material.authoring.auto_layout.PositionedGraphReportsZeroMoved` (movedCount 0, no transaction, undo queue length unchanged), `PinWright.material.authoring.auto_layout.UndoRestoresPositions` (latest undo entry is the auto_layout transaction; one UndoTransaction puts both back at (0,0)). Docs: wiki-src material.authoring.md `### material.authoring.auto_layout`, CHANGELOG. Not done here: the shared required `scope` parameter (tracked by the cross-verb contract in F-blueprint-auto-layout-rpc; adding a required param would break current callers) and fix item 3 (open Material Editor copy overwriting the moves) remains unverified.
