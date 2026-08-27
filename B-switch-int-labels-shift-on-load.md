---
id: B-switch-int-labels-shift-on-load
title: "BPIR `switch_int` case labels silently renumber on asset load when they are not a contiguous run from `StartIndex`"
status: OPEN
severity: High
category: bug
tags: [bpir, k2node-switchinteger, wrong-data-that-looks-correct]
---

# BPIR `switch_int` case labels silently renumber on asset load

Reproduced live, UE 5.8, plugin `d195a55d`, on a scratch Actor Blueprint.

Authored `switch_int(Index: 3) [1 -> @a, 2 -> @b, default -> @d]`, then read the node back:

| Point | Case pin names |
|---|---|
| After `compile_bpir` | `1`, `2` |
| After `asset.save` + `asset.reload` | **`0`, `1`** |

Same node id, same graph. The block authored for case `1` now sits on pin `0`, so selection `0` runs
case 1 and every arm is shifted by one. Nothing is logged.

**Control, same session:** authored `[0 -> @a, 1 -> @b, default -> @d]` — pins read `0`, `1` before and
`0`, `1` after the reload. Stable.

## Scope — this is conditional, not universal

The defect bites only when the authored labels are **not** the contiguous ascending run starting at the
node's `StartIndex` (default 0), in the order the exec targets appear in the BPIR text. With
`0,1,2,...` it is a no-op. That is why the wiki examples never expose it —
`wiki-src/bpir.examples.switch-int.md` and `bpir.instructions.md:131` both start at 0.

## Mechanism — the report's usual diagnosis is wrong

PinWright does **not** match case labels positionally. It creates each case pin *named* by the literal
label (`Compiler/BpirCompiler.cpp:5902-5920`) and wires by name (`FindExecOutputPin`, `:7653-7678`), which
is correct: for `UK2Node_Switch` the pin name **is** the case value
(`K2Node_Switch.cpp:373` `GetExportTextForPin` returns `Pin->PinName`, consumed at `:127` as a literal
term). The live read-back confirms this — pins came back as `1`, `2`.

The corruption is engine-side, on reconstruction.
`Editor/BlueprintGraph/Private/K2Node_SwitchInteger.cpp:99-126`
(`ReallocatePinsDuringReconstruction`) walks the old exec output pins in order and **renames them in
place** from a counter seeded at `StartIndex`:

```cpp
int32 ExecOutPinCount = StartIndex;
...
const FName NewPinName = GetPinNameGivenIndex(ExecOutPinCount);
ExecOutPinCount++;
TestPin->PinName = NewPinName;
```

`GetPinNameGivenIndex` (`:73-76`) is just `Printf("%d", Index)` — `StartIndex` is not added there.
`CreateCasePins()` is a deliberate no-op (`:93-97`) and `PinNames` is unused by the integer switch, so pin
identity across reconstruction is purely positional.

**When it fires:** not on the `compile_bpir` compile itself — that is an in-editor compile, so
`BlueprintCompilationManager.cpp:1331-1340` takes the `OptionallyRefreshNodes` branch. It fires on
**compile-on-load** (`if (BP->bIsRegeneratingOnLoad) FBlueprintEditorUtils::ReconstructAllNodes(BP)`,
`:1334`), and on a `PostEditChangeProperty` for `StartIndex`/`bHasDefaultPin`, and on a manual refresh.
Hence: correct in-session, corrupt on the next load — which is exactly why a decompile-diff taken right
after compiling looks clean.

## Contributing gaps

- `blueprint.graph.set_node_property` cannot set `StartIndex`. Its accepted set is a hardcoded whitelist
  of five (`Comment`/`NodeComment`, `X`/`NodePosX`, `Y`/`NodePosY`, `bCommentBubbleVisible`,
  `bCommentBubblePinned`) with no reflective fallback — `BlueprintGraphCrudHandler.cpp:1973-2047` returns
  `PROPERTY_NOT_SUPPORTED`. BPIR's `node_props { }` block is also rejected for the `switch_int` opcode
  (`BpirParser.cpp:1944-1950`).
- The decompiler never emits `StartIndex` (`Decompiler/BpirTextEmitter.cpp:1872-1905`), so round-tripping a
  node with a non-zero `StartIndex` silently resets it to 0.

## Suggested fix

Either author case pins as the contiguous run the engine will renumber to and carry the intended case
values elsewhere, or set `StartIndex` to the lowest authored label and require the labels to be contiguous
from it — rejecting non-contiguous label sets with a real error rather than compiling something that will
change meaning on load. Emitting `StartIndex` from the decompiler is needed either way.

## Coverage gap

`Tests/Compiler/TestCompilerIntegration.cpp:1394-1439` (`CompileSwitchInt`) and
`Tests/Bpir/TestBpirRoundTrip.cpp:1233-1291` (`round_trip.SwitchInt`) both use `[1 -> ..., 2 -> ...]` —
the exact failing shape — and assert only `bSuccess` plus "a node whose class name contains
SwitchInteger exists". Neither asserts pin names, and neither reloads. `Docs/bpir-test-matrix.md:54`
marks `switch_int` coverage "Full".

A test that compiles, reconstructs (or saves and reloads), and asserts the case pin names would catch it.

## History

- `#1-reported-with-live-repro` `OPEN` reporter — Confirmed at runtime on `d195a55d` / UE 5.8: labels `1,2` read back as `1,2` after compile and as `0,1` after save+reload; control with labels `0,1` stable across the same cycle. Engine reconstruction path verified in engine source; PinWright's by-name wiring confirmed correct by the pre-reload read.
