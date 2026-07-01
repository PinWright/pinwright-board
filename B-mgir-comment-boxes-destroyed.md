---
id: B-mgir-comment-boxes-destroyed
title: "MGIR round-trip silently destroys all UMaterialExpressionComment boxes — text, position, color, size lost without warning"
status: WONTFIX
severity: Medium
category: bug
tags: [mgir, decompiler, compiler, comment, data-loss]
---

# Comment boxes are invisible to MGIR — round-trip wipes them

`UMaterialExpressionComment` nodes are stored on the material in
`EditorComments[]`, separate from the regular
`ExpressionCollection.Expressions[]` array.

- The decompiler reads expressions via `Material->GetExpressions()`,
  which returns only `Expressions[]`. **Comment boxes are never
  visited.**
- The compiler's `ClearMaterialGraph()` (`MGIRCompiler.cpp:338`)
  empties `EditorComments` in Append mode.

Net result: every comment box (text, position, color, size,
group annotation) is **silently destroyed** on any MGIR
decompile→compile round-trip. No warning is emitted. The author's
organization disappears.

## Why this matters

Comment boxes are the primary in-editor mechanism for documenting
intent in large materials — labeling sections ("UV Setup",
"Lighting Pass", "World-Position Modifications"), marking TODOs,
crediting external authors. Losing them silently corrupts the
material's documentation layer.

## Fix

Decompiler: walk `Material->EditorComments[]` after the main
expression walk and emit each comment as a top-level MGIR
statement:

```mgir
comment "Text content here" @(x, y, w, h) color=(R=0.5, G=0.5, B=0.5, A=1)
```

Compiler: parse the `comment` statement, construct
`UMaterialExpressionComment`, set `Text`/`SizeX`/`SizeY`/`MaterialExpressionEditorX`/`MaterialExpressionEditorY`/`CommentColor`,
add to `EditorComments[]`. Adjust `ClearMaterialGraph()` to either
preserve `EditorComments` or replay them from the MGIR text.

## Repro

Pick any material with comment boxes (most non-trivial materials
have at least one). `material.decompile_mgir` → no `comment`
statements in output. `material.compile_mgir` of the same text →
all comment boxes gone from the material.

## History
- `#1-initial-spec` `OPEN` reporter — MGIR coverage parity audit confirmed `UMaterialExpressionComment` lives in `EditorComments[]`, separate from `ExpressionCollection.Expressions[]`. Decompiler doesn't walk it; compiler clears it on Append. Silent total data loss. Fix: add a top-level `comment "..." @(x,y,w,h) color=(...)` MGIR statement, decompile + compile both sides.
- `#2-wontfix-visual-only` `WONTFIX` developer — Out of scope. MGIR round-trips are guaranteed logically equivalent, not visually equivalent. Comment boxes are pure documentation/visual organization (no effect on compiled material), same class as node positions and graph layout — preserving them is a separate visual-fidelity goal that the IR doesn't promise. Reclassify to `F-mgir-visual-fidelity` if a future user explicitly asks for editor-side preservation.
