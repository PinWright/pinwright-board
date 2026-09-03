---
id: B-bpir-split-input-pin-args-unresolvable
title: "Split struct *input* pins decompile to per-sub-pin args (`NewLocation_X:`) no fresh node carries, so the graph does not recompile — and on a variable set the same blindness silently drops the wiring and emits a stale literal"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, decompile, compile_bpir, round-trip, split-pin, struct, sub-pin, silent-data-loss, write-side]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# The write side of the decompiler is split-struct-pin blind

`B-bpir-break-struct-pin-not-named` taught the decompiler to resolve split struct **output**
pins to dotted member paths (`$Hit.HitBoneName`, `%n0.OutHit.BoneName`). The **input** side
was left untouched and is blind in two different ways depending on whether the emit site
filters `bHidden`. Both break the decompile → edit → recompile round trip; one of them does
it silently.

## Mode A — sub-pin names leak into arg lists; recompile hard-fails

`UEdGraphSchema_K2::SplitPin` hides the struct pin (`EdGraphSchema_K2.cpp:7422`), creates one
sub-pin per member named `<ParentPinName>_<MemberName>` (`:7430`) with `ParentPin` set
(`:7461`), and inserts the sub-pins into `Node->Pins` immediately after the parent
(`:7525`).

`FormatArgs` (`Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp:779-812`) skips the
hidden parent at `:793` (`if (Pin->bHidden) continue;`) and has no `ParentPin` filter, so
every sub-pin is emitted as its own argument at `:801`, named with the raw sub-pin name.
Shipped output, from the committed dump mirror:

```
# asset-dumps/App/Blueprints/Pawn/PW_Crane/bpir.txt:16
call K2_SetRelativeLocation(Target: $SM_Motor, NewLocation_X: %n2,
    NewLocation_Y: $SM_Motor.RelativeLocation, NewLocation_Z: $SM_Motor.RelativeLocation,
    bSweep: false, bTeleport: false)
```

A freshly created `K2_SetRelativeLocation` node carries one unsplit `NewLocation` (FVector)
pin and no `NewLocation_X`. On recompile `ResolveTargetPin_Default`
(`Compiler/BpirCompiler.cpp:712-746`) fails every lookup — exact, case-insensitive and
space-normalized all miss — so `WireDataPins` takes the `!TargetPin` branch
(`:7157-7165`), logs `BuildMissingPinHint` (`:1620-1667`) and accumulates an error.
`AccumulatedErrors.Num() > 0` triggers the atomic rollback at `:3517-3525`, so the whole
compile fails and every created node is deleted. Loud, but a **hard blocker**: the graph
cannot be round-tripped at all.

The same shape reaches macro-output emission (`BpirTextEmitter.cpp:2245-2259`) and
`make_array` (`:2363-2372`) — both filter `!Pin->bHidden` and emit sub-pins — and the
`return` emitter (`:2199-2211`), which filters neither and therefore emits the hidden parent
**and** all of its sub-pins as sibling args.

Nested splits exist in real content and are doubly unresolvable:
`ToolTransform_Location_X`, `AppendTransform_Rotation_Pitch`.

## Mode B — the value is silently replaced by a literal and the wiring is dropped

`EmitVariableSet` (`BpirTextEmitter.cpp:2029-2064`) takes the **first** non-exec, non-self
input pin (`:2049-2058`) with no `bHidden` and no `ParentPin` guard. Because the engine
inserts sub-pins *after* the parent (`EdGraphSchema_K2.cpp:7525`), that first pin is the
hidden parent. The parent has no links (they all moved to the sub-pins) and `SplitPin` never
clears its `DefaultValue`, so `ResolveInputValue` falls through to the typed-default path
(`Decompiler/BpirDecompiler.cpp:2565-2581`) and `FormatPinDefaultLiteral`
(`:2042-2120`) returns a struct literal for any `PC_Struct` pin. The warning at `:2573-2578`
fires only for `<unresolved>`, which a struct pin never yields.

Net effect: `set MyLocation = 0.0, 0.0, 0.0` for a node whose X member is wired from a live
value. The IR reads as complete and correct; nothing warns; the upstream pure node is still
emitted by `EmitPureDependencies` (`:2586+`) as an orphan `%nN` nobody consumes. Recompiling
that IR writes a variable set fed by a stale literal — **silent loss of the authored
wiring**, the same class as the parent ticket, on the write side.

The identical unguarded first-input-pin loop also backs the switch selector
(`BpirTextEmitter.cpp:1907-1917`) and the `break<T>` struct input (`:2312-2322`). The switch
selector is effectively unreachable (no K2 switch takes a struct selector); a split input on
an explicit break node is reachable but unusual.

Mode B is only *silently* destructive when a graph's split pins are confined to variable
sets: any Mode-A arg elsewhere in the same compile fails the whole thing and rolls back
first. That is luck, not a guard.

## Minimal repro

1. In any Blueprint function, add `Set Actor Location`. Right-click its `New Location` pin →
   **Split Struct Pin**. Wire `New Location X` from a `Get Actor Location` → `Break Vector`
   `X` (or any float source); leave Y and Z at their defaults.
2. `blueprint.decompile` → the call emits `NewLocation_X: %n0.X, NewLocation_Y: 0.0, ...`.
3. Feed that same text back to `blueprint.compile_bpir` → fails with
   `Could not find target pin 'NewLocation_X' on node 'Set Actor Location'. Did you mean
   'NewLocation'? Available pins: ...`, and rolls back.

For Mode B, replace step 1 with a `Set` node on an FVector member variable and split its
value pin: the decompile emits `set MyVec = 0.0, 0.0, 0.0` with no warning, and the recompile
"succeeds" with the wiring gone.

## Reach (committed dump mirror, `asset-dumps/`)

`grep` over the 1,701 `bpir.txt` files for split-sub-pin argument names
(`<Name>_(X|Y|Z|Location|Normal|Rotation|Scale|Pitch|Yaw|Roll|R|G|B|A|Min|Max|W):`):

- **70 files** hit, **64 distinct assets** after collapsing the `/App`–`/Game` redirect mirror
  (~4% of dumped Blueprint graphs).
- **2,340 argument occurrences** total. Most common parents: `Location_*` (283 each axis),
  `Scale_*` (145), `SpawnTransform_*`, `NewRotation_*`, `NewLocation_*`,
  `RelativeTransform_*`, `InstanceTransform_*`.
- Concrete instances: `asset-dumps/App/Blueprints/Pawn/PW_Crane/bpir.txt:16,24`,
  `asset-dumps/App/Meshes/Truck/BP_Truck/bpir.txt:13`,
  `asset-dumps/App/PathTracer/Showcase/ShowcaseBlueprints/PC_DemoPlayerController/bpir.txt:218`.

Mode B has no text signature by construction (that is the defect), so it is not counted here.

## Documentation is now wrong

`Plugins/PinWright/docs/wiki-src/bpir.md:56-59` — added by the parent fix — states: *"A struct
pin that is **split** in the graph … decompiles to that same dotted form … The raw
`<ParentPin>_<Member>` sub-pin name never appears in the IR."* That is true of outputs and
false of inputs, on 2,340 shipped occurrences. An agent that trusts the sentence will not
recognise `NewLocation_X:` as a defect.

**Workaround:** before recompiling a decompiled graph that contains split input pins, hand-
rewrite each sub-pin arg group into a `make<T>` feeding the parent pin name:

```
%loc = make<Vector>(X: %n2, Y: $SM_Motor.RelativeLocation.Y, Z: $SM_Motor.RelativeLocation.Z)
call K2_SetRelativeLocation(Target: $SM_Motor, NewLocation: %loc, bSweep: false, bTeleport: false)
```

There is no workaround for Mode B, because nothing tells you the value was lost.

## Fix — two options

**Option 1 (preferred): emit `make<T>` on the decompile side.** Group a node's sub-pins by
split root, emit one `make<StructName>(Member: <value>, ...)` pure instruction ahead of the
consuming statement, and pass `%tmp` under the **parent** pin name. `make<T>` already exists
in the grammar (`docs/wiki-src/bpir.instructions.md:249`), so BPIR gains no new syntax and the
compiler needs no change — exactly the reasoning that picked dotted access over a split-pin
syntax on the read side. The sub-pin *values* are already correct after the parent fix; this
is purely a regrouping of existing value expressions. Must recurse for nested splits
(`ToolTransform_Location_X` → `make<Transform>(Location: make<Vector>(...))`). Mode B is fixed
by the same grouping plus a `bHidden`/`ParentPin` guard on the five unguarded first-input-pin
loops (`BpirTextEmitter.cpp:1907, 2049, 2199, 2312`, and `FormatArgs` at `:793`).
*Trade-off:* the recompiled graph gains an explicit Make node instead of a split pin — the
same cosmetic drift the read side already accepted (split break → explicit break node), and
`B-bpir-break-struct-emits-generic-node` (OPEN) warns that generic make/break nodes are
rejected for some native-break structs, so `make<Rotator>` may be blocked until that lands.

**Option 2: split the pin on demand in the compiler.** When an argument name matches
`<PinName>_<Member>` for an existing struct pin, call `UEdGraphSchema_K2::SplitPin` on the
parent and retry the lookup. Precedent exists in-plugin:
`Handlers/Blueprint/BlueprintGraphCrudHandler.cpp:1411-1428` and `:1828-1845` already mirror a
node's split state onto a replacement node this way. *Trade-off:* the recompiled graph is
byte-identical to the original (split preserved, no extra node), but it keeps the raw sub-pin
spelling in the IR — contradicting the documented rule at `bpir.md:56-59` and re-introducing
the near-synonym ambiguity the parent ticket closed (`Hit_BoneName` vs `Hit_HitBoneName` are
distinct, but the spelling is node-local and not resolvable by reading the IR alone). It also
does nothing for Mode B, which is a decompiler bug, not a compiler one.

Whichever option is taken, Mode B needs the guard regardless — a hidden parent must never be
mistaken for a node's value pin.

## Related

- `B-bpir-break-struct-pin-not-named` (IN-REVIEW) — parent. Fixed the read side; its Fix
  section names this gap as deliberately out of scope.
- `B-bpir-break-struct-emits-generic-node` (OPEN) — generic make/break nodes rejected for
  native-break structs; gates Option 1 for `Rotator` and friends.
- `E-decompile-bpir-text-not-verbatim-roundtrip` (IN-REVIEW) — the general "decompile is a
  re-emitter, not a transcript" gap; this is a case where the re-emission is not merely
  non-verbatim but non-compilable.
- `F-bp-graph-replace-node-rpc` — the split-mirroring code cited as Option 2's precedent.

## Fix

Root cause: input emission treated hidden split parents as ordinary pins in the
variable-set/first-input scans and emitted visible split children as raw named
arguments. Those names do not exist on a fresh unsplit node, while variable sets
selected the hidden parent's stale default and discarded child wiring.

Option 1 is implemented. `BpirTextEmitter.cpp` now recursively groups split input
children into deterministic standalone `make<Struct>(...)` values and passes the
parent name/value to calls, macros, returns, make containers, generic nodes, and
variable sets. Select options retain their false/true or enum names while using the
same grouping. The affected scans skip hidden parents/children; invalid or orphaned
split pins and invalid node GUIDs fail closed with the established `<unresolved>`
value and warning, and failed nested construction rolls back generated lines and
temporary-name state. `BpirDecompiler.cpp` adds authored positions to each generated
line. `AppendNodeLine` consumes the emitter's explicit generated-prefix count, so only pure
helpers use the existing compiler placement cadence (300 units left, then +80 and +130
vertical stacking); every fan-out consumer statement retains its authored position, and
reconvergence still maps to the prefix's first line. The split-pin contract
pages (`docs/wiki-src/bpir.md` and `bpir.instructions.md`) now distinguish generated input
makes from dotted/explicit-break output handling. `bpir.txt` aspect 7 was bumped to 8 and
its pinning test was updated.

Regression IDs:

- `PinWright.bpir.split_input.VectorFunctionCall`
- `PinWright.bpir.split_input.VectorVariableSet`
- `PinWright.bpir.split_input.VectorSelect`
- `PinWright.bpir.split_input.MalformedChild`

The function-call and variable-set tests exercise the production decompiler and
compiler in a decompile -> parse/compile round trip after their structural emitter
assertions. `VectorFunctionCall` keeps authored positions and checks that the generated
MakeVector is offset from its consumer.

Deliberate nonchanges: no compiler or verb contract change was needed because
standalone `make<T>(...)` already parses/compiles; Option 2 and unrelated generic
make/break support remain out of scope. No live editor, compile, or test run was
performed per the ticket instructions. No BPIR verb syntax changed; only its split-pin
decompile contract documentation was corrected.

## History
- `#1-filed` `OPEN` reporter — Found by code reading while reviewing the `B-bpir-break-struct-pin-not-named` fix, which closed the split-struct **output** path and explicitly deferred the input path. Confirmed against plugin and engine source, no editor. Two distinct modes: (A) `FormatArgs` (`Decompiler/BpirTextEmitter.cpp:779-812`, `bHidden` skip at `:793`, arg emit at `:801`) writes one arg per sub-pin under the raw `<Parent>_<Member>` name, which `ResolveTargetPin_Default` (`Compiler/BpirCompiler.cpp:712-746`) cannot resolve on a freshly created node, so `WireDataPins` (`:7157-7165`) errors and the atomic rollback at `:3517-3525` fails the whole compile — a hard blocker on the round trip; same shape in the macro-output, `make_array` and `return` emitters. (B) `EmitVariableSet` (`:2029-2064`, loop at `:2049-2058`) has no `bHidden`/`ParentPin` guard and takes the hidden parent as the value pin — the engine inserts sub-pins after the parent (`EdGraphSchema_K2.cpp:7525`) — so the wiring is dropped and a struct literal is emitted instead (`BpirDecompiler.cpp:2565-2581`, `:2042-2120`), with no warning: silent loss, same class as the parent ticket. Reach measured on the committed dump mirror: 2,340 sub-pin arg occurrences across 70 `bpir.txt` files / 64 distinct assets of 1,701 (~4%), e.g. `asset-dumps/App/Blueprints/Pawn/PW_Crane/bpir.txt:16`. The parent fix's own doc line (`docs/wiki-src/bpir.md:56-59`) now over-promises that the raw sub-pin name never appears in the IR. Fix proposed two ways — emit `make<T>` on the decompile side (no grammar or compiler change, matches the read side's design), or `SplitPin`-on-demand in the compiler (precedent at `Handlers/Blueprint/BlueprintGraphCrudHandler.cpp:1411-1428`) — with trade-offs in the Fix section; Mode B needs the hidden-parent guard either way.
- `#2-regroup-split-inputs` `IN-REVIEW` developer — Grouped split struct input children into standalone make values, guarded hidden-parent scans, added function-call/variable-set regressions, and bumped the BPIR aspect version to 8; live verification remains for the tester.
