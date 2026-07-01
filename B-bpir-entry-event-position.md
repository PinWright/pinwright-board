---
id: B-bpir-entry-event-position
title: "BPIR decompile omits entry event node position"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, compiler, round-trip, layout]
---

# BPIR decompile omits entry event node position

`blueprint.decompile` appends `@(x, y)` to node-backed instructions in an entry body, but the entry signature itself does not carry the entry node's graph position:

```bpir
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) {
    %n0 = call GetOwningPlayer() @(316, 1902)
    %n1 = call K2Node_AsyncAction(...) @(316, 1633)
}
```

The event node itself also has `NodePosX` / `NodePosY` in the UE graph, but BPIR drops it. Round-tripping this text can recreate or move the body nodes while leaving the event entry node to compiler defaults or auto-layout behavior.

## Repro

Decompile `/App/App/UI/W_AppMapEditor` from:

```text
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\UI\W_AppMapEditor\bpir.txt
```

Observed: both `CommonButtonBaseClicked__DelegateSignature` entries have positioned body nodes, but neither `entry event ...` line includes a coordinate suffix.

## Expected

BPIR should preserve entry node positions for event entry points. A deterministic round-trip form could be:

```bpir
entry event CommonButtonBaseClicked__DelegateSignature(object<CommonButtonBase> Button) @(16, 1902) {
    %n0 = call GetOwningPlayer() @(316, 1902)
}
```

The compiler must apply that coordinate to the created or reused entry event node. If the syntax is generalized, it should apply consistently to other entry node kinds that have visible graph nodes, such as `entry override`, `entry custom_event`, function entries, and macro tunnel entries.

## Required Fix

1. Extend the BPIR grammar/parser to allow optional `@(x, y)` on entry signatures before `{`.
2. Update decompiler entry-signature emission to append the source entry node's `NodePosX` / `NodePosY`.
3. Update compiler entry creation/reuse paths to apply the authored entry position.
4. Keep body-position validation coherent: entry position must not count as a body instruction position, and mixed body positioning rules should remain body-scoped.
5. Add a round-trip regression where an event node and at least one body node have distinct coordinates; decompile, compile, and verify both positions are preserved.

## History

- `#1-initial-report` `OPEN` reporter — Found in `W_AppMapEditor` BPIR dump. Body nodes include `@(x, y)`, but `entry event CommonButtonBaseClicked__DelegateSignature(...)` omits the event node position, so BPIR cannot fully preserve graph layout.
- `#2-entry-position-roundtrip` `IN-REVIEW` developer — Added authored positions on BPIR entry signatures, applied them to compiled entry nodes, emitted source entry node coordinates during decompile, and added parser plus round-trip regression coverage.
- `#3-verified` `DONE` tester — Decompiled `/App/App/UI/W_AppMapEditor`; entry signatures now emit `@(x, y)` before `{` (e.g. `entry widget_event MenuButton.OnButtonBaseClicked(...) @(0, 1633) {`, `entry override Init(...) @(0, 788) {`).
