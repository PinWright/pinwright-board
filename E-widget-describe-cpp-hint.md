---
id: E-widget-describe-cpp-hint
title: "`widget_describe` TREE_EMPTY should indicate C++ programmatic trees"
status: DONE
severity: ""
category: ergonomic
tags: []
---

# `widget_describe` TREE_EMPTY should indicate C++ programmatic trees

`widget_describe` returns `TREE_EMPTY` for C++ widgets that build trees programmatically (e.g., `UCircularMinimapWidget`). Can't distinguish "empty widget" from "C++ programmatic tree".

**Proposal:** Check if the widget's parent class overrides `Initialize()` or `NativeConstruct()` and report the C++ parent class name instead of a bare error.

## History
- `#1-tree-empty-repro` `OPEN` reporter — W_Minimap (extends UCircularMinimapWidget) returned TREE_EMPTY twice. Had to read C++ source to understand the tree.
- `#2-added-programmatic-check` `IN-REVIEW` developer — Before returning TREE_EMPTY, walks ParentClass to first native class. If native parent != UUserWidget, returns TREE_PROGRAMMATIC with C++ class name.
- `#3-verified-tree-programmatic` `DONE` tester — Verified: `mcp__editor_automation__.call path="widget.describe" args={...}` on W_Minimap (extends UCircularMinimapWidget) returns TREE_PROGRAMMATIC with message "parent class 'CircularMinimapWidget' likely builds its tree programmatically in C++".
