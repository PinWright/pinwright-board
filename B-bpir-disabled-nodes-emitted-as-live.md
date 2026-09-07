---
id: B-bpir-disabled-nodes-emitted-as-live
title: "blueprint.decompile / bpir.txt renders DISABLED nodes as live BPIR with no marker — a fully disabled EventGraph decompiles to three entry events calling parent, and the BPIR language has no syntax for node enabled-state at all"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, bpir-decompiler, blueprint-decompile, asset-dump, disabled-nodes, node-state, enabledState, lossy-ir, silent-wrong-data, round-trip, weapons]
encounters: 4
lastSeen: 2026-09-06T00:00:00Z
---

# The IR says the subclass overrides three lifecycle events; the graph says every node is compiled out

BPIR is the surface a reviewer reads to answer "what does this Blueprint do". For a disabled node it
answers wrong, and there is no token in the language that could answer right.

## What was called

```
asset.dump {assetPath: "/Game/FPS/Weapons/BP/BP_Weapon_AR"}      → bpir.txt
blueprint.graph.get_node_details {nodeId: "2FDCE57B482B8CD531FE198D476FAB95", ...}
```

## What happened — measured

`BP_Weapon_AR`'s **entire `EventGraph` is disabled**. `blueprint.graph.get_node_details` says so
unambiguously, on all six nodes:

```
nodeState.enabledState: "disabled"
nodeState.isEnabled: false
```

`bpir.txt` for the same asset emits **three live `entry event` blocks** — `BeginPlay`,
`ActorBeginOverlap`, `Tick` — each calling parent, with no marker, no comment, no warning, and
nothing in the `# ==== Warnings ====` block.

A reviewer reading only the IR concludes the subclass overrides those three events and chains to
the parent implementation. In fact every node in the graph is compiled out: the subclass overrides
nothing, and the parent's behaviour runs unmodified.

## The gap is in the language, not just this asset

`bpir.md` — the BPIR language reference, read at
`X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\bpir.md` — contains **no mention of
`disabled`, `enabledState`, or node state of any kind**. This is not a decompiler that forgot to
emit a token that exists; there is no token. The IR is structurally incapable of representing a
disabled node, which means:

- **Decompile is lossy in a way that inverts meaning.** Most IR omissions lose detail. This one
  turns "does nothing" into "calls parent".
- **Round-trip is lossy in the same direction.** A `compile_bpir` of that IR would author *live*
  nodes back over what were disabled ones — the decompile→edit→recompile loop silently enables a
  graph the author deliberately turned off.

## What was expected

The signal is already in hand at decompile time, on the same node objects the decompiler walks, and
the plugin already exposes it one layer down: `blueprint.graph.get_nodes` / `get_node_details` emit
a structured `nodeState` block (`isEnabled`, `enabledState`, `isDisabledByUser`) whenever
`includeNodeState: true`. Only BPIR drops it.

## What is asked for

1. **A BPIR token for node enabled-state**, and a line in the language reference defining it. The
   two engine states that matter are `disabled` (compiled out) and `disabled-and-not-passing-flow`
   vs `enabled-and-not-passing-flow` (the "development only" middle state) — at minimum, disabled
   must be distinguishable from enabled.
2. **`compile_bpir` must accept it**, or the round-trip re-enables what it reads. A read-only marker
   is strictly worse than none for the decompile→edit→recompile loop this IR exists to serve.
3. **Fallback if (1) is too large a language change for now: a warning.** A
   `# BPIR_WARN: Node emitted as live but is disabled in the graph: <graph> :: <class> '<name>' nodeId=<guid>`
   line per disabled node, in the existing warnings block, is a small diff and stops the IR from
   reading clean over a graph that is switched off. It does not fix the round-trip.
4. **Entry-level shorthand.** A graph where *every* node is disabled (this case) deserves to say so
   once at the top rather than per-node — a reviewer's first question about `BP_Weapon_AR` is "does
   this subclass do anything", and the answer is one line.

## Root cause — guess, no source read taken

The decompiler almost certainly walks nodes and emits statements without consulting
`UEdGraphNode::GetDesiredEnabledState()` / `IsNodeEnabled()`, the same accessors the graph-inspection
handler uses to build its `nodeState` block. **Inference from the two surfaces' behaviour; no plugin
source was opened for this ticket and no `file:line` is claimed.**

## Severity

**Medium**, sitting just under the High silent-wrong-data band. It is silent, it is wrong, and it is
on the normal path — `bpir.txt` is written by every `asset.dump` of every Blueprint. It is held at
Medium rather than High because a second published verb (`get_node_details` with
`includeNodeState: true`) does carry the truth, so the information is one call away for a caller who
thinks to doubt the IR — which is exactly the caller this defect does not produce. A fixer who
judges that "the IR reads as live over a compiled-out graph, and recompiling it enables the graph"
belongs in the High band will get no argument from this reporter.

## Related

- `E-inspect-events-omits-disabled-stub-flag` (IN-REVIEW, Medium) — **the same missing signal on the
  third surface.** `blueprint.inspect`'s `events[]` lists disabled default stubs indistinguishably
  from authored handlers; its fix adds an `enabled` boolean derived from `SourceNode->IsNodeEnabled()`
  in `CollectBlueprintEvents`. That ticket's own scope note says the graph-node layer "is already
  solved" via `nodeState` and narrows itself to `events[]` — leaving BPIR as the remaining surface,
  which is this ticket. The three should be consistent: `nodeState` (done), `events[].enabled`
  (in review), BPIR token (here).
- `B-bpir-decompiler-emitter-coverage-gap`, `B-bpir-delegate-signature-lost`,
  `B-bpir-composite-inline-name-lost`, `B-bpir-fallthrough-reconverge-dropped` — the existing family
  of BPIR fidelity losses. This one is distinguished by direction: those lose information, this one
  asserts information that is false.
- `B-bpir-orphan-warning-diagnostics-lossy` — the warnings block this ticket's fallback (3) would
  extend, and the precedent for BPIR emitting a per-node `# BPIR_WARN:` line with a nodeId.
- `B-orphan-finder-vs-decompiler-disagree` — the other case this review round found of two verbs
  disagreeing about the same graph, also on `/Game/FPS/Weapons/`.

## Fix

BPIR now carries the non-default Blueprint node enabled states as bare
`disabled` and `devonly` suffixes on entry signatures and body
instructions. `FBpirTextEmitter::GetNodeEnabledStateMarker` reads
`UEdGraphNode::GetDesiredEnabledState()` for decompilation; `FBpirParser`
accepts the markers with authored positions before or after the `@(x, y)`
marker; and `FBpirCompiler` applies the parsed state with
`SetEnabledState()` to both entry nodes and emitted instruction nodes. The
language reference defines the syntax, and direct transient parser plus
compile/decompile automation covers marker preservation in an in-memory graph.
That automation does not exercise the `blueprint.decompile` /
`blueprint.compile_bpir` handlers on a saved asset, and the generic call lane
still does not preserve `UK2Node_CallParentFunction` identity: the decompiler
emits it as `call` and the compiler creates a plain call node. Parent-call
round-trip coverage and a handler-backed saved-asset check remain required.
Test IDs:
`PinWright.bpir.parser.EnabledStateSuffix` and
`PinWright.bpir.round_trip.EnabledState`.

Changed files: `Source/PinWright/Private/Compiler/BpirTypes.h`,
`BpirSharedConstants.h`, `BpirParser.cpp`, `BpirCompiler.cpp`,
`Decompiler/BpirTextEmitter.{h,cpp}`, `Decompiler/BpirDecompiler.cpp`,
`Tests/Bpir/TestBpirEnabledState.cpp`, and
`docs/wiki-src/bpir.md` plus `bpir.entry-points.md`. No Unreal build, editor run, or live
asset verification was performed in this source-only pass.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. `BP_Weapon_AR`'s entire `EventGraph` is disabled — `blueprint.graph.get_node_details` on `2FDCE57B482B8CD531FE198D476FAB95` returns `nodeState.enabledState: "disabled"` and `isEnabled: false`, on all six nodes — yet the asset's `bpir.txt` emits three live `entry event` blocks (`BeginPlay`, `ActorBeginOverlap`, `Tick`), each calling parent, with no marker, no comment and nothing in the warnings block. A reviewer reading only the IR concludes the subclass overrides those three events and chains to the parent; in fact every node is compiled out and the subclass overrides nothing. The gap is in the language, not just the emitter: the BPIR language reference read at `X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\bpir.md` contains no mention of `disabled`, `enabledState`, or node state of any kind, so BPIR cannot represent a disabled node — which makes decompile lossy in a meaning-inverting direction ("does nothing" becomes "calls parent") and makes the round-trip lossy the same way, since a `compile_bpir` of that IR would author live nodes back over deliberately disabled ones. The signal is already in hand: the plugin's own `blueprint.graph.get_nodes` / `get_node_details` emit a structured `nodeState` block (`isEnabled`, `enabledState`, `isDisabledByUser`) under `includeNodeState:true`; only BPIR drops it. Ask, in order: a BPIR token for node enabled-state plus its definition in the language reference; `compile_bpir` acceptance of that token (a read-only marker would leave the round-trip re-enabling what it reads); as a smaller fallback, a per-node `# BPIR_WARN:` line naming each disabled node emitted as live, following `B-bpir-orphan-warning-diagnostics-lossy`'s precedent; and an entry-level shorthand for the all-nodes-disabled case, which is the first question a reviewer has about this asset. Root cause is a **guess**: the decompiler likely emits statements without consulting `GetDesiredEnabledState()` / `IsNodeEnabled()`, the accessors the graph-inspection handler already uses — inferred from the two surfaces' behaviour, no plugin source was opened and no `file:line` is claimed. Severity Medium, just under the High silent-wrong-data band: silent, wrong, and on the path every `asset.dump` of a Blueprint takes, but held down because `get_node_details` with `includeNodeState:true` carries the truth one call away — for a caller who thinks to doubt the IR, which is the caller this defect does not produce.
- `#2-roundtrip-measured-it-hard-fails-not-re-enables` `OPEN` WEAPONS — Ticket #1 predicted the round-trip would "author *live* nodes back over what were disabled ones". Measured on the same asset and the same nodeId (`2FDCE57B482B8CD531FE198D476FAB95`, `BP_Weapon_AR`): **it does not silently re-enable — it hard-fails**, and for a second, separate reason. Feeding the decompiled IR straight back returns `BLUEPRINT_COMPILE_FAILED: Function 'ReceiveBeginPlay' called from BeginPlay should not be called from a Blueprint; Function 'UserConstructionScript' called from Construction Script should not be called from a Blueprint`. Cause: the emitted `call ReceiveBeginPlay()` / `call UserConstructionScript()` render a **`K2Node_CallParentFunction`** as an ordinary unqualified `call`, and BPIR has no parent-call opcode, so the token round-trips into a plain function call the Kismet compiler rejects. So this asset carries TWO stacked losses on the same statement: the node is disabled (ticket #1) and it is a Parent-call rendered as a normal call (this entry). Both are meaning-inverting in the same direction — the IR claims the subclass chains to the parent; it neither chains nor runs. Note the emitter is inconsistent: once the event was enabled and given a real body, decompile switched to the qualified `call SKEL_BP_WeaponBase::ReceiveBeginPlay()` form (cf. `B-bpir-decompile-emits-skel-class-prefix-on-self-event-calls`), so the unqualified form appears to be what the ghost/disabled stub emits. Practical consequence for authors: `compile_bpir` on any child Blueprint whose entry chains to its parent is **destructive** — upsert deletes the Parent-call node and cannot recreate it. Working alternative found: `blueprint.insert_bpir_at_node` anchored on the `K2Node_CallParentFunction` node splices after it without touching it, and as a side effect UE promotes the ghost stub to `enabledState: "enabled"` (verified via `get_node_details` before/after) — which is the correct outcome here but is undocumented and silent. Adding to the ticket's ask list: BPIR needs a parent-call opcode, or `compile_bpir` must refuse rather than drop a `K2Node_CallParentFunction` it cannot re-emit. Encountered while re-rooting `BP_WeaponBase` and wiring magazine/slide components; three Blueprints authored, 0 orphans throughout.
- `#3-returned-still-live-plus-two-new-symptoms` `OPEN` WEAPONS-critic — **Returned: the defect is unchanged in use, and two new symptoms make it worse.** Status was already `OPEN` (no fix has been recorded on this ticket), so this is a return of evidence rather than a status flip; **severity raised Medium → High**, on the justification `#1` itself invited — see the end of this entry. Re-measured in a WEAPONS critic review round 3. **Core defect unchanged:** `blueprint.get` reports `BP_Weapon_AR`'s `ReceiveActorBeginOverlap` and `ReceiveTick` as `enabled: false` (so the `E-inspect-events-omits-disabled-stub-flag` fix has landed on *that* surface and the truth is now trivially in hand), while both `blueprint.decompile` and `bpir.txt` still emit them as live `entry event` blocks with **no marker and no warning** — neither ask (1) nor even the small fallback ask (3) has been acted on, and the two surfaces now contradict each other inside one session. **New symptom (a) — the parent-call qualifier is dropped exactly on the disabled events, and only on those.** Pistol: `call SKEL_BP_WeaponBase::ReceiveBeginPlay()` and `::ReceiveTick(…)` (both enabled, qualified) but a bare `call ReceiveActorBeginOverlap(…)` (disabled, unqualified). AR: `SKEL_BP_WeaponBase::ReceiveBeginPlay()` (enabled, qualified) but bare `call ReceiveTick(…)` and `call ReceiveActorBeginOverlap(…)` (both disabled). Across the two assets that is **4 disabled → 4 unqualified, 3 enabled → 3 qualified** — a perfect split, which turns `#2`'s "the unqualified form appears to be what the ghost/disabled stub emits" from a hunch into a measured rule. The consequence is worse than the compile failure `#2` recorded: round-tripping the AR's BPIR would compile `entry event Tick { call ReceiveTick(DeltaSeconds) }` — the event calling **itself**, i.e. **infinite self-recursion**, not a rejected call. **New symptom (b) — a third treatment of the same case inside one class family.** `BP_WeaponTestPawn` and `BP_WeaponBase` both carry disabled `ReceiveActorBeginOverlap` / `ReceiveTick`, and there the decompiler **omits them entirely**. So one class family produces three different renderings of "disabled event": emitted-as-live-and-qualified is absent, emitted-as-live-and-unqualified (AR, Pistol), and omitted-without-trace (TestPawn, WeaponBase). A reviewer cannot infer anything about a graph's disabled nodes from BPIR, in either direction — silence does not mean absent and presence does not mean live. **Severity:** raised to **High** on the silent-wrong-data band. `#1` held it at Medium because `get_node_details` carried the truth one call away; symptom (a) removes that mitigation, because the failure is no longer "a reviewer misreads the IR" but "a mechanical round-trip of the IR authors an infinitely self-recursive event", on the normal path, with nothing in the emitted text to prompt a check. `#1`'s own severity note said a fixer who judged this High would get no argument; the round-trip measurement is what decides it. Adding to the ask list: whatever token or warning lands must also fix the **qualifier**, since a disabled parent-call currently loses the one piece of information that makes it re-emittable. No plugin source was opened for this entry; the 4-vs-3 split, the recursion consequence, and the three-treatment inventory are all read off `blueprint.decompile` / `bpir.txt` / `blueprint.get` responses, and no `file:line` is claimed. Related and confirmed clean the same round: `B-orphan-finder-vs-decompiler-disagree` (now DONE — the finder and BPIR agree on 0 orphans across all four of these assets), so BPIR's disagreement with the graph verbs is now specifically about node *state*, not reachability.
- `#4-still-reproduces-round-four` `OPEN` WEAPONS-critic — Re-measured in a WEAPONS critic review round 4 on `/Game/FPS/Weapons/BP/BP_Weapon_AR`: **every symptom `#3` recorded still reproduces, and nothing on the ask list has been acted on.** `encounters` 3 → 4, **severity unchanged at High**, **status unchanged at OPEN** (no fix has ever been recorded on this ticket, so this is a fourth observation, not a return of a fix). Measured, verbatim: `bpir.txt` emits `entry event Tick(float DeltaSeconds) { call ReceiveTick(DeltaSeconds: $DeltaSeconds) }` as live, with the `SKEL_BP_WeaponBase::` qualifier dropped, while `blueprint.graph.find_nodes {includeNodeState: true}` on the same asset reads `K2Node_Event_2 "Event Tick"` as `enabledState: "disabled", isEnabled: false` and `K2Node_CallParentFunction_2 "Parent: Tick"` as `enabledState: "disabled", isEnabled: false`. Both halves of the statement — the entry event and the parent call it wraps — are disabled in the graph and live in the IR, with no marker, no comment and nothing in the warnings block. The pistol's **enabled** Tick still keeps its qualifier, so `#3`'s measured rule holds on a third round of data: **the qualifier drop correlates exactly with the disabled state**, and the correlation is now observed across three separate sessions rather than inferred from one. The consequence `#3` identified as the reason for the High rating is unchanged and was re-confirmed: a mechanical round-trip of the AR's BPIR still authors `entry event Tick { call ReceiveTick(...) }` — the event calling itself, **infinite self-recursion** — because the qualifier that would have made it a parent call is exactly what the disabled state drops. `ActorBeginOverlap` still shows the same treatment on both AR and pistol. Note for a fixer on where this now sits relative to its siblings: `blueprint.graph.find_nodes` with `includeNodeState` is a **third** verb carrying the truth (alongside `get_node_details` from `#1` and `blueprint.get`'s `enabled` flag from `#3`), so the signal is now available on three published surfaces and dropped by exactly one — which removes any remaining argument that the information is hard to reach at decompile time, and leaves the smallest fallback ask (3), a per-node `# BPIR_WARN:` line, without a cost justification for deferring it further. No plugin source was opened for this entry; everything above is read off `asset.dump` / `bpir.txt` and `blueprint.graph.find_nodes` responses, and no `file:line` is claimed.
- `#5-node-enabled-state-roundtrip` `IN-REVIEW` developer — Added `disabled` / `devonly` BPIR suffixes and parser fields for entry signatures and body instructions; decompilation now reads `GetDesiredEnabledState()`, and compilation replays the state with `SetEnabledState()` for entry and emitted nodes. Added the language reference plus direct transient parser and compile/decompile round-trip automation. The test bypasses the `blueprint.decompile` / `blueprint.compile_bpir` handlers and saved-asset persistence, and it does not cover `UK2Node_CallParentFunction`: the current generic call lane still decompiles a parent call as `call` and recompiles it as a plain call node, so parent-call identity remains a follow-up before the broader round-trip claim is complete. Source checks passed (`git diff --check` on the touched plugin files); no Unreal build, editor run, or live asset verification was performed, so tester verification is still required.
