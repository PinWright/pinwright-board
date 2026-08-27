---
id: B-bpir-make-array-loses-ftext-literals
title: "BPIR make_array writes literal FText elements into DefaultValue instead of DefaultTextValue, so they compile to empty text -- the same defect just fixed on select"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, k2node-makearray, ftext, wrong-data-that-looks-correct, DefaultTextValue]
encounters: 1
lastSeen: 2026-08-27
---

# The same misrouting `select` had, on the other node kind that can hit it

`B-bpir-select-literal-text-lost` is fixed: `select` now pre-types its option pins before applying
literals, so an `FText` literal reaches `DefaultTextValue` -- the slot `UK2Node_Select::ExpandNode`
actually reads -- instead of `DefaultValue`, where it compiled to empty text.

`make_array` has the identical shape. `UK2Node_MakeArray` element pins are `PC_Wildcard` at emit time
(`BpirCompiler.cpp`'s `EBpirOpcode::MakeArray` path resolves `[N]` pins), so
`make_array("NSLOCTEXT(...)", ...)` routes the literal to the wrong slot the same way and loses the
text. The BPIR source says the right thing, the compile succeeds, the node exists, and the text is
gone at runtime.

Bounded: `make_set` and `make_map` are not emitted at all (documented gaps), so `select` and
`make_array` are the only two node kinds on this path. `select` is done; this is the other one.

**Fix:** the same shape -- resolve the element pin type before pass 3a applies the literal, so
`FCodePinResolver::SetPinDefaultValue` takes its existing (already correct) text branch.

## History
- `#1-named-by-the-select-fix` `OPEN` reporter -- Named by the agent fixing
  `B-bpir-select-literal-text-lost`, which swept for other node kinds reaching the same wildcard
  escape hatch. Source-level claim; not reproduced.
- `#2-pre-type-make-array-pins` `IN-REVIEW` developer -- Confirmed by source reading and fixed the
  same shape as the select fix. `BpirCompiler.cpp` gains `ResolveMakeArrayOutputPinType` /
  `PreTypeMakeArrayPins`, called from the `EBpirOpcode::MakeArray` emit case before pass 3a applies
  element literals; the type comes from the `%name: array<T> =` annotation the decompiler already
  emits for every make_array, else from an element literal that parses as an FText with a real
  localization identity. The output `Array` pin is stamped too, which is load-bearing: it stops the
  engine re-propagating (and re-defaulting) the element pins when a later instruction wires the array
  away. `CodePinResolver.cpp` unchanged -- its PC_Text branch was already correct and simply never
  reached. New test `PinWright.bpir.round_trip.MakeArrayTextElementLiteral`
  (`Tests/Bpir/TestBpirMakeArrayTextElementLiteral.cpp`) asserts DefaultTextValue carries the display
  string, namespace and key with DefaultValue empty, on both the inference and the annotation path,
  plus a negative leg that a bare quoted element is not mistyped as text. `Docs/bpir-test-matrix.md`
  rows updated and gap 24 appended. Not compiled or run -- the orchestrator owns builds.
