---
id: B-import-xml-replace-moves-subtree-to-end
title: "`widget.import_xml` replace with `targetName` re-inserts the rebuilt subtree as the LAST child of its parent, silently reordering Horizontal/VerticalBox layouts"
status: OPEN
severity: High
category: bug
tags: [widget, import-xml, replace, child-order, horizontal-box, vertical-box, silent-wrong-result]
encounters: 1
lastSeen: 2026-09-23T19:20:00Z
---

# "Preserves its parent slot" keeps the slot values but not the slot's position

`widget.import_xml {widgetPath: /App/App/UI/LobbyAndMenu/Elements/W_AppUserPanel, targetName: "LessonBlock",
xml: "<SizeBox name=\"LessonBlock\" ...>...</SizeBox>"}` returned `success: true`. `LessonBlock` was child
index 0 of `HB_Top` (HorizontalBox: `[LessonBlock, SizeBox_0]`). After the import it was index 1
(`[SizeBox_0, LessonBlock]`): the PIE screenshot showed the profile block on the left and the lesson block
on the right. The slot's `Padding` (`Right=-12`) did survive, so the "parent slot preserved" claim in the
wiki is half true.

Same result for `targetName: "NoPupilsPanel"` in `W_AppSchoolNameLogin`: the panel moved from index 1 of
the `LoginBox` VerticalBox (above the name fields) to the last index (below the anonymous button).

Nothing in the response says the index changed. For box panels the index *is* the layout, so this is a
silent wrong result that only a visual check catches.

**Workaround:** follow every replace with `widget.reparent_widget {widgetName, newParent: <same parent>,
placement: {index: <old index>}}` (this keeps the slot padding).

**Fix (proposed):** record `Parent->GetChildIndex(Old)` before removal and insert the new root at that
index (`UPanelWidget::InsertChildAt` / `ReplaceChild`), as `widget.wrap` and `widget.replace_class`
already do via `ReplaceChild`.

## History
- `#1-replace-appends-at-end` `OPEN` reporter - Filed from a UMG pass on UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Two subtree replaces (`LessonBlock` in `W_AppUserPanel`, `NoPupilsPanel` in `W_AppSchoolNameLogin`) both landed as the last child of their box parent; caught only by a PIE screenshot; fixed with `widget.reparent_widget placement.index`.
